source "https://rubygems.org"

# Pinned to the versions that produce the current published site so that
# generated URLs remain byte-for-byte identical during the migration.
# These can be bumped when the theme is updated (step 2).
gem "jekyll", "~> 3.10"

# _config.yml uses kramdown with GFM input.
gem "kramdown-parser-gfm", "~> 1.1"

group :jekyll_plugins do
  gem "jekyll-paginate", "~> 1.1"
  gem "jekyll-feed", "~> 0.17"
end

# Required on Ruby 3.x for `jekyll serve` (WEBrick was removed from stdlib).
gem "webrick", "~> 1.8"

# Gems that were removed from Ruby's default set in 3.4 but are needed by
# jekyll 3.10 and its dependencies (e.g. safe_yaml requires base64).
gem "base64"
gem "bigdecimal"
gem "csv"
gem "logger"
