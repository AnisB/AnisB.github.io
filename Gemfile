source "https://rubygems.org"

# Local preview. GitHub Pages builds the site with its own pinned Jekyll version
# (the github-pages gem); the plugins below are on the GitHub Pages allow list.
gem "jekyll", "~> 4.4"
gem "webrick"

group :jekyll_plugins do
  gem "jekyll-sitemap"
end

# Windows needs these for time zone support.
platforms :windows do
  gem "tzinfo"
  gem "tzinfo-data"
end
