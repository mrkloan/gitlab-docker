# gitlab-docker
> [Dockerized GitLab setup](https://docs.gitlab.com/omnibus/docker/README.html) including *Mattermost*, a *Docker registry* and a GitLab CI *Docker Runner* for your CI pipelines

## Docker

Install `docker` using:

```bash
$ sudo curl https://get.docker.com | bash
```

Add a non-root user to the `docker` group:

```bash
$ sudo usermod -aG docker <USER>
```

Install `docker-compose` using:

```bash
$ sudo curl -L https://github.com/docker/compose/releases/download/1.14.0/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
$ sudo chmod +x /usr/local/bin/docker-compose
```

## Gitlab setup

The repository structure is as follows:

```txt
docker-compose.yml
gitlab-ce/
├── config/
├── data/
└── logs/
gitlab-runner/
README.md
```

Run the containers and check that they are effectively running using:

```bash
$ docker-compose up -d
$ docker ps # 2 containers should be running: gitlab-ce & gitlab-runner
```

If you're hit by an error `500` or `502` when trying to access the application, try reloading the page or updating the shared directories permissions:

```bash
$ docker exec -it gitlab-ce update-permissions
```

### GitLab Community Edition

A bunch of files should have been created into `gitlab-ce` subdirectories.

Define a DNS name for this GitLab instance by editing `gitlab-ce/config/gitlab.rb`:

```ruby
# gitlab-ce/config/gitlab.rb
external_url 'http://<GITLAB_FQDN>'
```

If you want to enable HTTPS on your instance, place your `.key` and `.crt` files into a `gitlab-ce/config/ssl/` folder. If your external url is `gitlab.example.com`, then your files must be named `gitlab.example.key` and `gitlab.example.crt`:

```ruby
# gitlab-ce/config/gitlab.rb
external_url 'https://<GITLAB_FQDN>' # HTTPS
nginx['redirect_http_to_https'] = true

# If your certificate files are named differently, you can override their name here
nginx['ssl_certificate'] = "/etc/gitlab/ssl/<SSL_CERTIFICATE>.crt"
nginx['ssl_certificate_key'] = "/etc/gitlab/ssl/<SSL_CERTIFICATE>.key"
```

Once you're done editing the file, reconfigure GitLab and reload the server. Your GitLab server should now be accessible through the external URL you configured, given that the DNS redirects to your Docker host.

```bash
$ docker exec -it gitlab-ce gitlab-ctl reconfigure
$ docker exec -it gitlab-ce gitlab-ctl restart
```

### Mattermost
> https://docs.gitlab.com/omnibus/gitlab-mattermost/

Update `gitlab-ce/config/gitlab.rb` in order to configure your *Mattermost* server:

```ruby
# gitlab-ce/config/gitlab.rb
mattermost_external_url 'http://<MATTERMOST_FQDN>' 

# GitLab as the only external authentication source
mattermost['email_enable_sign_up_with_email'] = false
mattermost['email_enable_sign_in_with_email'] = false

# Configure an e-mail address and SMTP server for Mattermost
mattermost['email_send_email_notifications'] = true
mattermost['email_smtp_username'] = "<SMTP_USERNAME>"
mattermost['email_smtp_password'] = "<SMTP_PASSWORD>"
mattermost['email_smtp_server'] = "<SMTP_FQDN>"
mattermost['email_smtp_port'] = "<SMTP_PORT>" # 587
mattermost['email_connection_security'] = 'TLS' # 'TLS', 'STARTTLS' or nil
mattermost['email_feedback_name'] = "GitLab Mattermost"
mattermost['email_feedback_email'] = "<EMAIL_ADDRESS>"

# E-mail batching allowing users to control how often they receive notifications
mattermost['service_site_url'] = 'http://<MATTERMOST_FQDN>:80' # With protocol AND port
mattermost['email_enable_batching'] = true
```

If you want to enable HTTPS on your Mattermost instance:

```ruby
# gitlab-ce/config/gitlab.rb
mattermost_external_url 'https://<MATTERMOST_FQDN>' # HTTPS

mattermost_nginx['redirect_http_to_https'] = true
mattermost_nginx['ssl_certificate'] = "/etc/gitlab/ssl/<SSL_CERTIFICATE>.crt"
mattermost_nginx['ssl_certificate_key'] = "/etc/gitlab/ssl/<SSL_CERTIFICATE>.key"
mattermost['service_use_ssl'] = true
```

Once you're done editing the file, reconfigure GitLab and reload the server (see previous section). Your Mattermost server should now be accessible through the external URL you configured, given that the DNS redirects to your Docker host.

### Container registry
> https://docs.gitlab.com/ce/user/project/container_registry.html

In order to activate the GitLab container registry under the *same* domain as your GitLab instance, you just have to configure a new port for it to start listening on:

```ruby
# gitlab-ce/config/gitlab.rb
registry_external_url 'https://<GITLAB_FQDN>:<REGISTRY_PORT>'
```

The registry *REQUIRES* HTTPS in order to work properly. In that case, it will reuse the SSL certificate used by your GitLab instance, and simply bind on a new port (`REGISTRY_PORT`) to start listening to requests.

If you want to bind the registry to another FQDN, you will have to generate another SSL certificate and place it into the `gitlab-ce/config/ssl/` directory:

```ruby
# gitlab-ce/config/gitlab.rb
registry_external_url 'https://<REGISTRY_FQDN>:<REGISTRY_PORT>'

registry_nginx['ssl_certificate'] = '/etc/gitlab/ssl/<REGISTRY_CERTIFICATE>.pem'
registry_nginx['ssl_certificate_key'] = '/etc/gitlab/ssl/<REGISTRY_CERTIFICATE>.key'
```

Once you're done editing the file, reconfigure GitLab and reload the server.

## Gitlab Runner
> https://www.sheevaboite.fr/articles/installer-gitlab-ci-moins-5-minutes-docker

Get your server's runners registration token by navigating to `Admin area > Runners` (`/admin/runners`) and write it down.

Start the registration process of your GitLab CI Runner using:

```bash
$ docker exec -it gitlab-runner gitlab-runner register
Running in system-mode.

Please enter the gitlab-ci coordinator URL (e.g. https://gitlab.com/ci):
https://<GITLAB_FQDN>/ci
Please enter the gitlab-ci token for this runner:
<RUNNERS_REGISTRATION_TOKEN>
Please enter the gitlab-ci description for this runner:
[f5bb1e7dbd6c]: Docker Runner
Please enter the gitlab-ci tags for this runner (comma separated):
docker
Registering runner... succeeded                     runner=<RUNNER_ID>
Please enter the executor: shell, ssh, virtualbox, docker+machine, docker-ssh+machine, docker, docker-ssh, parallels:
docker
Please enter the default Docker image (eg. ruby:2.1):
java:latest
Runner registered successfully. Feel free to start it, but if it's running already the config should be automatically reloaded!
```

Reload the `Runners` administration page on your GitLab server; the newly created runner should have appeared!

Please note that the jobs using this runner will be running in *containers*, which will not be linked to the other 2 containers (`gitlab-ce` and `gitlab-runner`). Which means that the CI containers will try to resolve your GitLab instance FQDN using a DNS server (such as 8.8.8.8, or the one of your entreprise private network): your `/etc/hosts` file will not be used in this case! Be sure that your GitLab instance is reachable outside of your local-machine networks (localhost, VM, container, ...) if you want this runner to successfully build your projects.