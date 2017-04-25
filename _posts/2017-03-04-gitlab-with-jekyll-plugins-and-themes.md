---
title: Github Pages with Jekyll Plugins and Theme
tags: [blog, continuous integration]

---

Github allows us to host a Jekyll page using [Github Pages](https://pages.github.com/),
but the support for plugins and themes is very limited.
We cannot freely use the plugins that you want,
and we have exactly the same problem with the themes.
Only [safe plugins](https://jekyllrb.com/docs/plugins/) and
the next [available themes](https://pages.github.com/themes/) can be used.

At the beginning, I was happy with those limitations.
Plugins can make your life much easier sometimes
but I didn't need them at the moment.
So I installed [minimal mistakes](https://mmistakes.github.io/minimal-mistakes/)
theme *manually* and everything works.


![OctoJekyll](/assets/images/octojekyll.png){: .align-center}

The day I wanted to modify and to update the theme
I realized that I need a simpler alternative to work with.
It turned out to be much easier than I supposed at beginning.
Github Pages also allows us to upload an static page
instead of the source of a Jekyll page and we will
use this feature to our goal.

If you have minimal mistakes theme like me,
probably you can find useful the [migration to gem version](https://mmistakes.github.io/minimal-mistakes/docs/quick-start-guide/) as first step.

## Deploy manually to Github Pages

The simplest solution I found is to work with two different branches:

- `source` will contain the *website source code*, which the current master branch is
- `master` will contain only the *Jekyll output*, the `_site` folder in Jekyll project.

We will have the two branches working together in the same directory structure.
To do that will execute the follow commands,
but **be careful:**
the commands will remove the history of the `master` branch,
although it will be kept in the `source` branch.

```bash
# create the source branch from the master
git checkout -b source
# push this branch to git repository
git push

# build the jekyll site
bundle exec jekyll build
# move to the output directory
cd _site
# create a new git repository in this folder
git init .
# we commit everything in this folder, which is the page
git add .
git commit -m "Initial commit"
# push this commit to github pages, all the history to avoid conflicts
git remote add origin git@github.com:YOURUSERNAME.github.io
git push -u --force
```

Like the `_site` directory is ignored by git because `.gitignore` excludes it,
we can create another git repository to be used only with the `master` branch.

The `source` branch, where we will be working most of the time,
it will be host in the root of the directory.
We can work in it like we were working before in the `master`.

To deploy new changes we only have to:

```bash
bundle exec jekyll build
cd _site
git add .
git commit -m "New version"
git push
```

But also we can create `bin/deploy` file with execution permission and the following content:

```bash
#!/usr/bin/env bash
set -e # halt script on error

bundle exec jekyll build
cd _site
rm -rf .git
git init .
git add .
git commit -m "New version"
git remote add origin git@github.com:YOURUSERNAME.github.io
git push -u --force
```

We avoid conflicts forcing the push at the expense of don't have `master` branch history.
I don't find it very useful anyway.

I found this solution very simple to work with and very integrated.
Another advantage is that we can have our `source` and published site desynchronized,
i.e: we can be working in a new draft post and push it to the `source` branch,
so will be saved in Github repostory
but until we don't push the `master` branch will not be published.

On the other hand,
if we forgot by mistake to push the `master` branch our work will never be published.
We can automate the *deploy* process using Travis CI,
or we can, in fact, use both methods at the same time.

## Testing Jekyll site

[`html-proofer`](https://github.com/gjtorikian/html-proofer)
is a gem very useful to test the output of our Jekyll site.
We add to the `Gemfile`:

```ruby
gem "html-proofer" # Add this line
```

Run `bundle install` to be installed.

And we create a new script, `bin/test`, to check the html generated is correct:

```bash
#!/usr/bin/env bash
set -e # halt script on error

bundle exec htmlproofer --allow-hash-href  --internal-domains YOURDOMAIN ./_site
```

To test it we need the execute the Jekyll server previously.

```
$ bin/test
Running ["LinkCheck", "ScriptCheck", "ImageCheck"] on ["./_site"] on *.html...


Checking 30 external links...
Ran on 8 files!


HTML-Proofer finished successfully.
```

No problem in this output! :dizzy:


## Deploy automatically to Github Pages with Travis-ci and CI

If you don't like manual deployment we have another free alternative.
[Travis CI](https://travis-ci.org) is a platform for continuous integration.
It allows us to build, test and deploy almost any project.
It can be used for free for open sourced projects
and we'll use this possibility for our web site.

![Travis CI](/assets/images/travis-ci.png){: .align-center}

We have to connect our Github page with the Travis one and
to add our Github pages repository.

After that we need those two files:

* `bin/build`: just to build the project easier

  ```bash
  #!/usr/bin/env bash
  set -e # halt script on error

  bundle exec jekyll build
  ```


* `.travis.yml`: this is the configuration for Travis CI
  ```yaml
  language: ruby
  env:
    global:
    - NOKOGIRI_USE_SYSTEM_LIBRARIES=true

  before_script: ./bin/build
  script: ./bin/test


  deploy:
    provider: pages
    github-token: $PRIVATE_TOKEN
    target-branch: master
    local-dir: _site
    project-name: blog
    email: YOUREMAIL@gmail.com
    name: Deployment bot

    skip_cleanup: true
    on:
      branch: source
    ```

Remember to add executable permissions to the files in the bin directory:

```bash
chmod +x bin/*
```

We make use of the
(private token of Github)[https://help.github.com/articles/creating-a-personal-access-token-for-the-command-line/]
for public repositories.
We add it to (variables in repository settings)[https://docs.travis-ci.com/user/environment-variables/#Defining-Variables-in-Repository-Settings]
like `PRIVATE_TOKEN`.

If everything is ok,
every time we push the source branch
Travis will build, test and push the new version to the Github pages.
If something fails it will send us an email with the error.
