# Main Blog updates

The goal of this project is to improve the build process for a github pages blog built using Jekyll. The blog currently has it's source files living in /mnt/home/Blog/raesene.github.io/ which then goes through a build process using `jekyll build -d ~/Development/raesene.github.io/` and then that directory is a repository synchronized to Github at https://github.com/raesene/raesene.github.io .

There are a couple of problems with this that I'd like to resolve

 - Mostly for jekyll sites, we just maintain the markdown in the repository, sync to github and it manages building to HTML and publishing. This site isn't like that for historical reasons, but we'd like to bring it in-line with that standard process
 - Currently the site uses the `master` branch in the repo. and we'd like the site to use `main` as is the modern convention.
 - Older content which was imported from previous blogs is present in the source repo as .html files for historic reasons. Whilst it's not a problem to leave them in that format, any new process will need to take account of them and ensure that they're published properly.
 

 In terms of constraints, it's very important that all URLs stay the same for all pages of the site as some of these are linked to.

 Also once this work is complete, we'd like to update the theme used, as a second step.