# frozen_string_literal: true

source "https://rubygems.org"

ruby ">= 3.1"

gem "jekyll-theme-chirpy", "~> 7.3", ">= 7.3.1"

group :test do
  gem "html-proofer", "~> 5.0"
end

platforms :mingw, :x64_mingw, :mswin, :jruby do
  gem "tzinfo", ">= 1", "< 3"
  gem "tzinfo-data"
end

gem "wdm", "~> 0.2.0", :platforms => [:mingw, :x64_mingw, :mswin]
