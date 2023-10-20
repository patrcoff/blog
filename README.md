# blog
personal blog written in markdown and generated using Hugo

### CI/CD

GH Action to deploy on PR plans below:

- needs to use ssh and rsync
- needs to auto-add host to known-hosts (nothing is secret in this repo, it's a public static site blog so there is no security risk here but should be remembered)

** git checkout performed before calling script **

```
apt install hugo
hugo
$SSH_KEY > ~/.ssh/gh_action
chmod 400 ~/.ssh/gh_action
rsync -are "ssh -o StrictHostKeyChecking=no -i $HOME/.ssh/gh_action" ./public/ $SSH_USER_AND_ADDRESS:public_html/blog/

```

