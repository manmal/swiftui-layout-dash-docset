source "https://rubygems.org"

# To run the site locally:
#   1. Install Jekyll: 
#        gem install bundler jekyll
#   2. Update bundled gems:
#        bundle install
#   3. Serve the site with drafts visible locally:
#        bundle exec jekyll serve --drafts
#   4. Point the browser at: http://localhost:4000
#
# For gems and versions supported by github-pags, see https://pages.github.com/versions/
gem "github-pages", group: :jekyll_plugins

# Plugins
group :jekyll_plugins do
  gem "jekyll-feed", "~> 0.12"
end

# Windows and JRuby does not include zoneinfo files, so bundle the tzinfo-data gem
# and associated library.
platforms :mingw, :x64_mingw, :mswin, :jruby do
  gem "tzinfo", "~> 1.2"
  gem "tzinfo-data"
end

# Performance-booster for watching directories on Windows
gem "wdm", "~> 0.1.1", :platforms => [:mingw, :x64_mingw, :mswin]

gem "webrick", "~> 1.7"
