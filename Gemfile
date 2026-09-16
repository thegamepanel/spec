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

# Jekyll renders a Markdown file as a page only when it has front matter. The
# documents all have it; README.md, INDEX.md and PROCESS.md do not, and were
# copied through as files rather than rendered. This renders them.
gem "jekyll-optional-front-matter", "~> 0.3"

# Gives a page a layout when it does not name one. ArchDoc's front matter has
# no layout key, so without this every document renders as a bare fragment
# with no theme around it.
gem "jekyll-default-layout", "~> 0.2"

# Takes a page's title from its first heading when it has none. The documents
# all carry a title in their front matter, but INDEX.md and PROCESS.md have no
# front matter at all, so without this the front door has a blank title.
gem "jekyll-titles-from-headings", "~> 0.5"

# The theme, and the two plugins it depends on.
gem "jekyll-theme-primer", "~> 0.6"
gem "jekyll-seo-tag", "~> 2.0"
gem "jekyll-github-metadata", "~> 2.9"
