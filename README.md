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
$ docker ps # 3 containers should be running: gitlab-ce, gitlab-runner, gitlab-registry
```

If you're hit by an error `500` or `502` when trying to access the application, try reload the page or updating the shared directories permissions:

```bash
$ docker exec -it gitlab-ce update-permissions
```

### GitLab Community Edition

A bunch of files should have been created into `gitlab-ce` subdirectories.

Define a DNS name for this GitLab instance by editing `gitlab-ce/config/gitlab.rb`:

```ruby
# gitlab-ce/config/gitlab.rb
external_url 'http://<GITLAB_FQDN>/'
```

Once you're done editing the file, reconfigure gitlab and reload the server. Your GitLab server should now be accessible through the external URL you configured, given that the DNS redirects to your Docker host.

```bash
$ docker exec -it gitlab-ce gitlab-ctl reconfigure
$ docker exec -it gitlab-ce gitlab-ctl restart
```

### Mattermost
> https://docs.gitlab.com/omnibus/gitlab-mattermost/

```ruby
# gitlab-ce/config/gitlab.rb
mattermost_external_url 'http://<MATTERMOST_FQDN>/' 

# GitLab as the only external authentication source
mattermost['email_enable_sign_up_with_email'] = false
mattermost['email_enable_sign_in_with_email'] = false

# Configure an e-mail address for Mattermost
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

*TODO*: e-mail configuration

Once you're done editing the file, reconfigure gitlab and reload the server (see previous section). Your Mattermost server should now be accessible through the external URL you configured, given that the DNS redirects to your Docker host.

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

Please note that the jobs using this runner will be running in *containers*, which will not be linked to the other 3 containers. Which means that the CI containers will try to resolve your GitLab instance FQDN using a DNS server (such as 8.8.8.8, or the one of your entreprise private network): your `/etc/hosts` file will not be used in this case! Be sure that your GitLab instance is reachable outside of your local-machine networks (localhost, VM, container, ...) if you want this runner to successfully build your projects.

### Registry

*TODO*