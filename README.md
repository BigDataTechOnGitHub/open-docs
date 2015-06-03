Mesosphere's Website
=======

### Deploy the site

Deployment of the website is done automatically, but to prevent a catastrophe there's a few limitations:

1. If you commit to the master branch, it only goes to [staging](http://docs-staging.mesosphere.com.s3-website-us-west-2.amazonaws.com/).
2. To move your changes to production, you need to merge from the master branch to the prod branch.
3. We use github pull requests to do review of changes before they go live, so don't do merging on the command line.

To get your changes into the staging website, do this:

    git checkout -b BRANCHNAME

Change BRANCHNAME to some short description of your changes.  Now do your changes and work like normal. When it's ready:

    git commit -am "MESSAGE"

Replace MESSAGE, with a description of your changes.  It has to have "" around it.  If you want to do an individual file, then it's slightly different:

    git commit -m "MESSAGE" FILE1 FILE2

Just use -m instead of -am and list each file to commit.

Finally, you do this to get your changes, in your branch, onto github:

    git push origin BRANCHNAME

Remember, BRANCHNAME is the name of your branch.

Once you've done that then you [Create a pull request](https://github.com/mesosphere/mesosphere-docs/compare/prod...master) online to get it reviewed.  To do the pull request do this:

1. On the left pulldown pick the master branch.
2. On the right, choose your branch, or use the list that shows up below and click on your branch.
3. Review your changes, and if it looks good, then click the green "Create pull request" button in the top left.

After this gets merged in (either by you or someone else), view your changes at http://docs-staging.mesosphere.com.s3-website-us-west-2.amazonaws.com/ to see the staging server.  If it's not right then simply make more changes and do another pull request.

### Handling Images

Images are actually stored on cloudfront, so to put an image on the website do this:

1. Do your branch and everything like normal.
2. Put your image in \_assets/img/
3. git add \_asset/img/MYIMAGE.jpg
4. Do your commit and push like normal.

That puts your image online and then you need to add this to your .md file to reference it:

    <img class="img-responsive" src="{% asset_path MYIMAGE.jpg %}" title="A TITLE" alt="" itemprop="image">

Be sure to set the "A TITLE" part and replace MYIMAGE.jpg with your file *without the \_assets/img/ part*.

Once you've done all that continue with your usual process to commit this, push it, and do a pull request.


### Run it locally

1. Install [RubyGems](https://rubygems.org/pages/download)
2. Install Bundler

        sudo gem install bundler
3. Add the default gems from the root of the project (you may need libxml2 - see [here](http://nokogiri.org/tutorials/installing_nokogiri.html) for instructions)

        bundle install --path vendor/bundle
4. Run Jekyll from the root of the project

        # Run Jekyll and watch for changes
        bundle exec rake dev
5. View the site in your browser: [http://localhost:4000](http://localhost:4000)

#### Troubleshooting

* ##### "invalid switch in RUBYOPT -S (RuntimeError)"

    This occurs when running `bundle install`.

    **The fix**: The path to your local repository cannot contain spaces. Rename any
    folders in the path to remove spaces.

### Add a new post

Posts live in the `_posts` directory and can be written in HTML or in Markdown.
Posts' filenames must always be this format:

    YEAR-MONTH-DAY-title.MARKUP

The date and title are used to generate the post's URL. For example, a post file
named `2013-08-01-how-to-eat-a-hotdog.html` would have this URL:

    /2013/08/01/how-to-eat-a-hotdog/

More details: [Writing posts in Jekyll](http://jekyllrb.com/docs/posts/).

#### Add Yourself as an Author

When authoring a post, you can add your name and Twitter handle in the
["authors.yml"](https://github.com/mesosphere/website/blob/master/_data/authors.yml)
file and reference the handle in the post.

For example:

**authors.yml**:

    bobmarley:
      name: Bob Marley
      twitter: reggaeman

**new-post.md**:

    ---
      ...
      author: bobmarley
      ...
    ---

    ...


#### Linkchecker

A weekly Dockerized Chronos task running on the OVH cluster runs the [linkchecker](http://wummel.github.io/linkchecker/) tool. The Chronos job description is in `utils/linkchecker.json` which runs a Python script that calls out to the `linkchecker` program and emails the output to the docs@mesosphere.io group.
