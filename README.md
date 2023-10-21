# blog
personal blog written in markdown and generated using Hugo

### CI/CD

GH Action to deploy on tagged commits; plans below:

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

### Content

**Remember!** When creating content, don't do it manually, use `hugo new content posts/filename.md`.

### Cloning

When cloning the repo ensure you use the `--recurse-submodules` switch.

To update the submodule you would run `git submodule update`

Not sure really how frequently that would be needed of course, it's only for the theme which likely doesn't git updated too often.