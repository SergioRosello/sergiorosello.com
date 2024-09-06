# Personal website

# Writing a post

This hugo site is configured in a way that my content is independent from the themes it may be used with.
All I have to do to add a post is:

1. Navigate to project working dir.
1. run command `hugo new content container-lifecycle-vulnerability-management/index.md`

In this example, a dir. called container-lifecycle.vulnerability-management will be created with a index.md file inside.

# Updating the theme

The theme is updated in a different manner.
Go to the `themes/<theme>` directory and `git pull`

# Uploading changes to prod

To deploy, use the hugo command, like so: `hugo deploy --target production`
