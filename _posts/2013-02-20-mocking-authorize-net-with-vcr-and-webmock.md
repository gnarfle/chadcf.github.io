---
layout: post
title: Mocking Authorize.net with VCR and webmock
created: 1361396177
---
Recently I was faced with making some fairly major changes to a Transaction class for a project, with exactly 0 tests written for it. This class is responsible for taking in information on an order (products, customer, etc), billing the customer via authorize.net and saving the appropriate entries to the database. It's a bit heavy and not exactly an ideal class, but it works and the goal is to now modify the way it works without breaking anything. Obviously, having tests is a good idea here.

The problem comes with the fact that the test is entirely dependent on an external service, in this case, authorize.net. As the goal is to test the functionality of the Transaction class, having tests success or fail depending on that 3rd party service means our tests are also testing authorize.net's api. This is not so great for isolating the functionality we want to test, plus it also makes our tests slow. On top of that, it makes it difficult to test giving partial refunds since authorize.net requires a 24 hour settlement before a refund can be given.

A great solution to this problem is using <a href="https://github.com/vcr/vcr">VCR</a> and <a href="https://github.com/bblimke/webmock">WebMock</a>. WebMock on it's own is a very capable library, which essentially allows you to mock HTTP requests. Almost anything your app does which involves a remote http request will be intercepted by WebMock, which will return whatever response you want. This is nice, however, manually filling out XML or YAML files to list the response to complicated API requests like Authorize.net is less than ideal.

Enter VCR. When you use VCR, either in your tests, other code or just in IRB, VCR will use WebMock to monitor any HTTP requests and will then save them to a .yml file for you. The next time the code is run, VCR will notice the response file already exists and just use that. In other words, it automates building WebMock responses for you. Though VCR supports other libraries in addition to WebMock, it seems WebMock is the preferred option so that is what I used.

One caveat before we begin: <b>Do not install webmock under your development or production dependencies!</b> Webmock by deafult intercepts all HTTP requests and returns a warning if it has no response defined. This means any HTTP requests without a webmock response defined will fail, which is not so pleasant.

Now we may begin. Let's look at the existing test and how we want to handle this. In this example I'm using Minitest for the testing, but VCR should work with any testing framework. I've also simplified the code somewhat to make what we're doing easier to understand. And, of course, I'm also using FactoryGirl for building stubs here.

<code>
require_relative '../spec_helper'

describe Transaction do
  before do
    @card = FactoryGirl.build_stubbed :card
    @customer = FactoryGirl.build_stubbed :customer
    @product = FactoryGirl.build_stubbed :product
    @items = { @product => 3 }
  end

  describe "when creating a new order" do
    it "should create an order" do
      transaction = Transaction.new(
        customer: @customer,
        items: @items,
        card: @card
      )
      transaction.execute!.wont_equal false
    end
  end
end
</code>

This should, hopefully, be pretty straight forward. Stub out some parameters required by the Transaction class using FactoryGirl (to isolate them from what we are testing), create a new transaction with details that should work, and verify it works. The problem is, this will actually go out and call authorize.net and if we're running a lot of tests, could get us rate limited or could just fail due to other network issues, not to mention will be quite slow.

Now let's switch to using VCR. Since in this case the only thing I really have external HTTP requests on is the Transaction class, I'm sticking the VCR configuration in this test file. You could also put it in your spec helper if you want it to be available across all tests.

First, install VCR and WebMock:

<b>Gemfile</b>
<code>
group :test do
  gem "vcr"
  gem "webmock"
end
</code>

and run <code>bundle install</code>.

Next, update our test file to use VCR/

<b>spec/lib/transaction_spec.rb</b>
<code>
require_relative '../spec_helper'
require 'vcr'

VCR.configure do |c|
  c.cassette_library_dir = 'spec/vcr_cassettes'
  c.hook_into :webmock
end

describe Transaction do
  before do
    @card = FactoryGirl.build_stubbed :card
    @customer = FactoryGirl.build_stubbed :customer
    @product = FactoryGirl.build_stubbed :product
    @items = { @product => 3 }
  end

  describe "when creating a new order" do
    it "should create an order" do
      transaction = Transaction.new(
        customer: @customer,
        items: @items,
        card: @card
      )

      VCR.use_cassette('authorize_success') do
        transaction.execute!.wont_equal false
      end
    end
  end
end
</code>

As you'll see, there are only two small changes here.

<code>
require 'vcr'

VCR.configure do |c|
  c.cassette_library_dir = 'spec/vcr_cassettes'
  c.hook_into :webmock
end
</code>

At the beginning of our test we require vcr and set two configuration options. The first is the cassette_library_dir, which is where vcr will store our http responses. In this case I've chosen to store them in the spec directory. Secondly the hook_into parameter, which tells vcr what http mocking library to use, which in this case is webmock.That's it!

The other change is to wrap any code that calls remote services in a vcr block.

<code>
      VCR.use_cassette('authorize_success') do
        transaction.execute!.wont_equal false
      end
</code>

In this case we're telling vcr to use a saved response named 'authorize_success' for any http requests in the following block of code. The first time you run this test, vcr will save the http response to the file spec/vcr_cassettes/authorize_success.yml. Subsequent requests will look for that file and use it, rather than calling authorize.net. That's all there is to it!

In reality, you'll probably need to generate multiple cassettes. For this test I have created responses that succeed or fail for purchases, refunds and voids and use them when appropriate in tests. If some of these are difficult to generate in a test, you can also do it in IRB using your development database (I had to do this to get the refund responses since they required transactions > 24 hours). 
