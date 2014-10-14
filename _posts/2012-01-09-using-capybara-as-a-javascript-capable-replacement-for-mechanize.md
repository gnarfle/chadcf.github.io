---
layout: post
title: Using capybara as a javascript capable replacement for Mechanize
created: 1326144202
---
Recently I had a task come to me, to update a content scraper that was written years ago with Ruby and <a href="http://mechanize.rubyforge.org/">Mechanize</a>. Unfortunately in the interim, the site had been completely revamped and was useless without javascript. Even though mechanize has been updated over the years, it still does not support javascript. This left me in something of a quandry... So I though, well, I do testing of rails apps that is essentially like screen scraping, and I have used a driver that supports javascript...

Could the combo of <a href="https://github.com/jnicklas/capybara">Capybara</a> and <a href="https://github.com/thoughtbot/capybara-webkit">Capybara Webkit</a> act as a replacement for Mechanize? 

The answer? Yes. At least for what I need. They both use <a href="http://nokogiri.org/">Nokogiri</a> underneath so the parsing should be pretty similar. 

Without further ado I present a simple script demonstrating how to write a command line screen scraper using capybara. This sample script can be run from the terminal and will perform a google search.

<code>
#!/usr/bin/env ruby 
require "rubygems"
require "bundler/setup"
require "capybara"
require "capybara/dsl"
require "capybara-webkit"

Capybara.run_server = false
Capybara.current_driver = :webkit
Capybara.app_host = "http://www.google.com/"

module Test
  class Google
    include Capybara::DSL
    
    def get_results
      visit('/')
      fill_in "q", :with => "Capybara"
      click_button "Google Search"
      all(:xpath, "//li[@class='g']/h3/a").each { |a| puts a[:href] }

    end
  end
end

spider = Test::Google.new
spider.get_results
</code>

This is a simple scraper that will perform a google search and spit out the links. I'm also using the following gemfile:

<code>
source :rubygems
gem "capybara-webkit"
gem "rake"
gem 'activerecord', :require => 'active_record'
gem "mysql2"
gem 'resque'
gem 'rspec'
gem 'database_cleaner'
gem 'launchy'
</code>

Though these may not be required depending on your project. 

Some tips:

<ul>
<li>Capybara webkit is tricky, because you can't see what is going on. I've done a fair bit of debugging using capybara with selenium, however selenium and capybara-webkit operate quite differently and often times something that works in selenium might not work in webkit. I've done a fair bit of "save_and_open_page" commands to figure this kind of thing out...</li>
<li>Javascript behavior is tricky.  Selenium offers some better control over waiting for elements to appear than capybara-webkit, however even that isn't perfect. Though often times capybara will properly wait for an ajax request to return, I've found on several instances I need to throw in a sleep to handle things properly (especially when dealing with something like a jquery modal popup).</li>
<li>My experience has been that xpath selectors are more reliable than css selectors, though also more complex...</li>
<li>Browser strings make a difference. In the above script, it would be different if using the selenium driver. The user agent capybara-webkit sends by default actually causes google to render different search result output, and disable the feature to start showing search results as you type. If you change the user agent with a command like:
<b>      page.driver.header("User-Agent", "Mozilla/4.0 (compatible; MSIE 8.0; Windows NT 5.2; Trident/4.0; .NET CLR 1.1.4322; .NET CLR 2.0.50727; .NET CLR 3.0.04506.30; .NET CLR 3.0.04506.648; MS-RTC LM 8; .NET CLR 3.0.4506.2152; .NET CLR 3.5.30729)")</b>
the above script will break. This can get tricky...
</ul>

This isn't a trivial process, however it gets the job done and it's not like using mechanize is trivial either. However, adding in the variables of delayed javascript actions as well as potentially different feature sets being delivered based on server side detection of browser features, it can get pretty complex. However, as far as I'm aware, this is about the only way out there to fairly easily scrape a javascript powered website.
