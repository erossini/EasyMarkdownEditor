# Contributing

Thank you so much for your interest in contributing to [PureSourceCode](https://puresourcecode.com/) EasyMarkdown Editor in different ways:

- add new PRs in [GitHub](https://github.com/erossini/EasyMarkdownEditor)
- ask questions or send ideas and improvements to me on [GitHub](https://github.com/erossini/EasyMarkdownEditor/issues)
- talk to my on my [Forum](https://puresourcecode.com/forum/)

## Coding

### Prerequisites

To contribute code to this project you'll need an up-to-date LTS or current version of Node.js and npm.

Please find information about the installation on [the official Node.js website](https://nodejs.org/en/download/).

### Workflow

Please make sure any code you submit is compliant and compatible with this repository's [license](./LICENSE).

#### Your first pull request

1. [Create a fork](https://github.com/erossini/EasyMarkdownEditor/fork) of this project.
1. Clone your fork: `git clone https://github.com/YOUR_USERNAME/easy-markdown-editor.git`.
1. Add the original repository as remote to keep it up-to-date: `git remote add upstream https://github.com/erossini/EasyMarkdownEditor.git`.
1. Fetch the latest changes from upstream: `git fetch upstream`.
1. Run `npm ci` to install the required dependencies.
1. Create a new branch to work on: `git checkout -b MyNewFeatureName`.
1. Write your awesome improvement and commit your work.
1. Make sure your changes comply with the established code and tests succeed: `npm run test`.
1. Push your changes to GitHub: `git push origin`.
1. On GitHub, go to your forked branch, and click **New pull request**.
1. Choose the correct branches, add a description and submit your pull request!

#### Continuing development

To create more pull requests, please follow the steps below:

1. Go back to the master branch: `git checkout master`.
1. Fetch the upstream changes: `git fetch upstream`.
1. Update the master branch with upstream changes: `git merge upstream/master`.

Thank you! 💜