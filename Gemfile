source "https://rubygems.org"

# Required on Ruby 3.x (no longer in stdlib)
gem "webrick", "~> 1.8"

# Production: GitHub Pages builds with this gem server-side. Skipped locally
# (see BUNDLE_WITHOUT below) because its jekyll-remote-theme dep fails on
# this machine's OpenSSL/CRL setup.
group :production do
  gem "github-pages", group: :jekyll_plugins
end

# Local preview: plain Jekyll 3.x + just-the-docs gem. No theme fetch.
group :development do
  gem "jekyll", "~> 3.10"
  gem "just-the-docs"
end
