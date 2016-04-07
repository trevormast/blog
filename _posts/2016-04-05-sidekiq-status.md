---
layout: post
title: "Progress Bar with SidekiqStatus and Sinatra"
date: 2016-04-05 17:22:04 -0800

---
Recently, I added a [Sidekiq][sidekiq-gh] worker to Jekyll.Pizza to break out services from HTTP requests.  Although it afforded a more graceful UI and snappier responses, I had to figure out how to keep the user updated on the status of their blog.  Using a progress bar seemed like a natural way to do this, but querying for updates on the worker was harder than I thought it would be. Fortunately I found [sidekiq_status][sidekiq_status-gh], which did most of the leg work.

# Setting Up SidekiqStatus
SidekiqStatus is an extension to Sidekiq that lets you query for information on your workers.

To start, add the gem to your Gemfile (I pulled the gem from GitHub because RubyGems didn't give me the right version):

{% highlight ruby %}
  gem 'sidekiq_status', git: 'https://github.com/cryo28/sidekiq_status.git'
{% endhighlight %}

 Then include the SidekiqStatus module in your Sidekiq worker class (I'll assume you're familiar with Sidekiq):

{% highlight ruby %}
class MyWorker
  include SidekiqStatus::Worker

  def perform(args)
    ...do something...
  end
end
{% endhighlight %}

When you call your worker, it will return a unique identifier:

{% highlight ruby %}
job_id = MyWorker.perform_async(args) ===> "fasdfjklfjajk4j455252jk"
{% endhighlight %}

Which can then be used to load a SidekiqStatus container:

{% highlight ruby %}
@container = SidekiqStatus::Container.load(job_id)
{% endhighlight %}
<!-- put container example here -->


# Cool Title 2
Somehow, I had to expose the container information to javascript.  I looked into [Gon][gon], which appears to be widely used in Rails, but the [Sinatra version][gon-sinatra] didn't implement the method I needed to keep updated on the state of the SidekiqStatus container.

Since my application was small--with only one worker--I decided to expose a JSON route to be polled for container data with AJAX.

First, I defined the route (which expects a job_id parameter) with Sinatra:

{% highlight ruby %}
get '/worker_status.json' do
  # '/worker_status.json?job_id="asdfs4dfs56df7sdf"'

  @job_id = params['job_id']

  @container = SidekiqStatus::Container.load(@job_id)

  @container.reload

  content_type :json
  { worker_status: "#{@container.status}" }.to_json
end
{% endhighlight %}

Then I made some necessary variables available to javascript:

{% highlight slim %}
/create.slim

javascript:

  BASE_URL = "#{root_url}"
  var job_id = "#{@job_id}"
  var json_url = BASE_URL + "worker_status.json?job_id=" + job_id;
{% endhighlight %}

Now our javascript has access to a url that will return JSON about the proper Sidekiq worker. We can then start polling for changes:

{% highlight slim %}
#./views/create.slim

javascript:

  BASE_URL = "#{root_url}";
  var job_id = "#{@job_id}";
  var json_url = BASE_URL + "worker_status.json?job_id=" + job_id;

  var intervalID = setInterval(function(){
    $.ajax({ url: json_url, success: function(data){
        var status = data['status'];

        document.getElementById("blog-status").innerHTML = status;

        #stops polling when status is complete
        if(data['worker_status'] === 'complete'){
          document.getElementById("blog-status").innerHTML = "Complete!"
          clearInterval(intervalID);
        }
    }, dataType: "json"});
  }, 1000); #polls every 1000ms or 1s

{% endhighlight %}

The above code uses an AJAX call to asynchronously query our JSON route and update the HTML on the page.  At this stage, all it does is update an h1 tag with the status of the worker (ie: 'waiting', 'working', 'failure', 'complete').  This doesn't tell the user much information.  Idealy, we would provide the user with an idea of how long the process will take. Again, SidekiqStatus makes it easy for us with the 'at' method.

{% highlight ruby %}
class MyWorker
  include SidekiqStatus::Worker

  def perform(args)
    self.total = 100 # default is 100
    at(0, 'Starting')
    ...do something...

    at(30, 'Doing Something')
    ...do something...

    at(60, 'Doing Something Else')
    ...do something....

    at(90, 'Doing Another Thing')
    ...do something...

    at(100, 'Finished!')
  end
end
{% endhighlight %}

Now, not only can we query for the status of the Sidekiq worker, we can also set custom messages that further describe the state of the worker process.  This makes it much easier to implement our progress bar.

First we'll implement a simple progress bar:

{% highlight slim %}
/create.slim

  #myProgress
    #myBar
      h1#blog-status Waiting...

{% endhighlight %}

{% highlight css %}
/* main.css */

#myProgress {
  position: relative;
  border-radius: 5px;
  width: 50%;
  height: 30px;
  background-color: grey;
}

#myBar {
  position: absolute;
  border-radius: 5px;
  width: 1%;
  height: 100%;
  background-color: green;
}
{% endhighlight %}

Then we'll ensure our JSON route will return the information we need:

{% highlight ruby%}
  ...
  content_type :json
  { worker_status: "#{@container.status}", at: "#{@container.at}", message: "#{@container.message}" }.to_json
end
{% endhighlight%}

And we'll access and update the new values in our AJAX call:

{%highlight ruby%}
var intervalID = setInterval(function(){
    $.ajax({ url: json_url, success: function(data){
        var status = data['status'];
        var message = data['message']
        var at = data['at'];

        document.getElementById("blog-status").innerHTML = message
        document.getElementById("myBar").style.width = at + "%"

        if(data['worker_status'] === 'complete'){
          document.getElementById("blog-status").innerHTML = "Complete!"
          clearInterval(intervalID);
        }
    }, dataType: "json"});
  }, 1000);
{% endhighlight %}

There you have it! Your site will now poll for JSON and update the progress bar every second.

[sidekiq-gh]: https://github.com/mperham/sidekiq
[sidekiq_status-gh]: https://github.com/cryo28/sidekiq_status
[gon]: https://github.com/gazay/gon
[gon-sinatra]: https://github.com/gazay/gon-sinatra
