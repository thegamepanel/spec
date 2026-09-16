# The site is built by .github/workflows/pages.yml rather than by Pages
# itself, so the gems it uses are declared here rather than supplied by the
# github-pages bundle.
#
# Jekyll 4 is the reason for the workflow: it can be told not to run Liquid
# over the documents, which Jekyll 3 cannot, and the documents contain Twig
# examples that Liquid reads as tags of its own.

source "https://rubygems.org"

gem "jekyll", "~> 4.4"

# Rewrites the relative links between documents, such as ../adr/0001-foo.md,
# to the pages they become. Pages supplies this by default and cannot disable
# it; here it has to be asked for.
gem "jekyll-relative-links", "~> 0.8"

# The theme, and the two plugins it depends on.
gem "jekyll-theme-primer", "~> 0.6"
gem "jekyll-seo-tag", "~> 2.0"
gem "jekyll-github-metadata", "~> 2.9"
