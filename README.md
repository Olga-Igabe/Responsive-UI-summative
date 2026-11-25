# EduFind - Educational Resource Finder

## Submission Links

- **Live site:** https://18-134-248-165.sslip.io
- **Demo video:** ADD YOUR VIDEO LINK HERE
- **GitHub repository:** https://github.com/Olga-Igabe/Responsive-UI-summative

## 1. Project Description

EduFind is an application for students. It helps them find books and learning materials of different subjects.

The application uses the Open Library API to retrieve book information and display it to users.

## 2. Problem Statement

Students sometimes find it difficult to discover educational books and learning materials.

EduFind provides a simple platform where students can search for books and explore learning resources.

## 3. Features

- Search for educational books.
- Search by title, author or keyword.
- Filter resources by subject.
- Sort books alphabetically.
- Sort books by publication year.
- Display book titles, authors, publication years and covers.
- View additional book information through Open Library.
- Handle API errors and empty search results.
- Search results are cached on the server for 10 minutes.
- Secure HTTPS connection.
- Responsive design for mobile and desktop devices.

## 4. Technologies Used

- HTML5, CSS3, JavaScript
- Python, Flask, Gunicorn
- Nginx (Web01, Web02)
- HAProxy (Lb01)
- Let's Encrypt (certbot) and sslip.io for HTTPS
- Open Library Search API
- Git and GitHub

## 5. External API

This project uses the Open Library Search API.

- API documentation: https://openlibrary.org/dev/docs/api/search
- Website: https://openlibrary.org/
- **API key:** none needed. Open Library is a free, public API.

Open Library provides public book information including titles, authors, publication years and covers.

Credit is given to the Open Library project for providing the book data used in EduFind.

### Application endpoints

| Endpoint | Purpose |
|---|---|
| `GET /api/books?q=<search>&sort=relevance\|title\|newest\|oldest` | Search books through Open Library |
| `GET /health` | Health check used by the load balancer |

## 6. How It Works

EduFind is an Educational Resource Finder that helps students discover books and learning materials using the Open Library API. Users access the application through their browsers, and their requests pass through LB01, which distributes traffic between Web01 and Web02. Both servers host the same application and handle requests to retrieve, search, sort and filter educational resources. This architecture improves traffic distribution, application availability and reliability.

```
Browser --HTTPS--> Lb01 (HAProxy) --> Web01 (Nginx -> Gunicorn -> Flask) --+
                                  --> Web02 (Nginx -> Gunicorn -> Flask) --+--> Open Library API
```

## 7. How to Run Locally

```powershell
git clone https://github.com/Olga-Igabe/Responsive-UI-summative.git
cd Responsive-UI-summative
py -m venv venv
venv\Scripts\Activate.ps1
pip install -r requirements.txt
python app.py
```

Then open http://127.0.0.1:5000 in your browser.

## 8. Deployment

| Server | Hostname | IP address |
|---|---|---|
| Web01 | 7286-web-01 | 3.9.117.161 |
| Web02 | 7286-web-02 | 13.40.94.133 |
| Lb01 | 7286-lb-01 | 18.134.248.165 |

### 8.1 Web01 and Web02

The same steps were done on both web servers.

```bash
ssh -i ~/.ssh/school ubuntu@3.9.117.161
sudo apt-get update
sudo apt-get install -y python3-venv python3-pip nginx git curl nano
git clone https://github.com/Olga-Igabe/Responsive-UI-summative.git edu-finder
cd ~/edu-finder
python3 -m venv venv
venv/bin/pip install -r requirements.txt
```

`/etc/systemd/system/edu-finder.service`

```ini
[Unit]
Description=Educational Resource Finder (Gunicorn)
After=network.target

[Service]
User=ubuntu
Group=ubuntu
WorkingDirectory=/home/ubuntu/edu-finder
ExecStart=/home/ubuntu/edu-finder/venv/bin/gunicorn --workers 3 --bind 127.0.0.1:5000 app:app
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now edu-finder
curl http://127.0.0.1:5000/health
```

`/etc/nginx/sites-available/edu-finder`

```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;

    add_header X-Served-By $hostname always;

    location / {
        proxy_pass http://127.0.0.1:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

```bash
sudo ln -sf /etc/nginx/sites-available/edu-finder /etc/nginx/sites-enabled/edu-finder
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl restart nginx
curl -sI http://localhost/health
```

### 8.2 Lb01 (load balancer and HTTPS)

```bash
ssh -i ~/.ssh/school ubuntu@18.134.248.165
sudo apt-get install -y haproxy certbot
sudo systemctl stop haproxy
sudo certbot certonly --standalone -d 18-134-248-165.sslip.io --agree-tos -m o.igabe@alustudent.com --non-interactive
sudo systemctl start haproxy
sudo mkdir -p /etc/haproxy/certs
sudo bash -c 'cat /etc/letsencrypt/live/18-134-248-165.sslip.io/fullchain.pem /etc/letsencrypt/live/18-134-248-165.sslip.io/privkey.pem > /etc/haproxy/certs/edu-finder.pem'
sudo chmod 600 /etc/haproxy/certs/edu-finder.pem
```

`/etc/haproxy/haproxy.cfg` (added parts)

```haproxy
global
    tune.ssl.default-dh-param 2048

frontend http_front
    bind *:80
    bind *:443 ssl crt /etc/haproxy/certs/edu-finder.pem
    http-request redirect scheme https code 301 unless { ssl_fc }
    http-request set-header X-Forwarded-Proto https if { ssl_fc }
    default_backend http_back

backend http_back
    balance roundrobin
    option httpchk GET /health
    http-check expect status 200
    server 7286-web-01 3.9.117.161:80 check
    server 7286-web-02 13.40.94.133:80 check
```

```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl restart haproxy
```

### 8.3 Testing

```
$ curl -sI http://18-134-248-165.sslip.io | head -3
HTTP/1.1 301 Moved Permanently
content-length: 0
location: https://18-134-248-165.sslip.io/

$ for i in 1 2 3 4; do curl -s https://18-134-248-165.sslip.io/health; echo; done
{"server":"7286-web-01","status":"ok"}
{"server":"7286-web-02","status":"ok"}
{"server":"7286-web-01","status":"ok"}
{"server":"7286-web-02","status":"ok"}
```

HTTP visitors are redirected to HTTPS, and requests alternate between Web01 and Web02.

## 9. Error Handling

The application handles:

- Failed API requests.
- Internet connection problems.
- Empty search results.
- Missing book information.
- Missing book covers.
- Open Library taking too long to answer (8-second timeout).
- Invalid searches (fewer than 2 or more than 100 characters).

## 10. Challenges

One challenge I faced was styling the Explore Resources button because the CSS changes were not appearing correctly in the browser. I fixed this by checking the HTML class name, updating the CSS selector and refreshing the browser.

I also encountered Git line-ending warnings when uploading my files to GitHub, but I learned that they were not errors and did not affect my code.

Another challenge was understanding how the load balancer distributes requests between the two application servers, which I addressed by studying the deployment architecture.

## 11. Future Improvements

- Add favorite books.
- Add more subject categories.
- Improve search performance.
- Add advanced filtering options.

## 12. Credits

- [Open Library](https://openlibrary.org/) for the book data and covers
- [Flask](https://flask.palletsprojects.com/), [Gunicorn](https://gunicorn.org/), [Nginx](https://nginx.org/), [HAProxy](https://www.haproxy.org/)
- [Let's Encrypt](https://letsencrypt.org/) and [sslip.io](https://sslip.io/) for HTTPS
- AI (Claude) was used for debugging.

## 13. Author

Olga Igabe, African Leadership University (o.igabe@alustudent.com)

Developed as a Summative assignment of Web Infrastructure.
