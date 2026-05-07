source "https://rubygems.org"

# Run Jekyll with `bundle exec`, e.g.
#
#     bundle exec jekyll serve
#
gem "jekyll", "~> 4.3"
gem "jekyll-theme-midnight"

# Required on Ruby 3+ (no longer in stdlib)
gem "webrick", "~> 1.8"

group :jekyll_plugins do
  gem "jekyll-feed", "~> 0.17"
  gem "jekyll-seo-tag", "~> 2.8"
  gem "jekyll-sitemap", "~> 1.4"
  gem "jekyll-paginate", "~> 1.1"
end

# Windows does not include zoneinfo files, so bundle the tzinfo-data gem
gem "tzinfo-data", platforms: [:mingw, :mswin, :x64_mingw, :jruby]

# Performance-booster for watching directories on Windows
gem "wdm", "~> 0.1.1", platforms: [:mingw, :mswin, :x64_mingw]

# JRuby/Linux file watching
gem "http_parser.rb", "~> 0.6.0", platforms: [:jruby]
