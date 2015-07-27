RethinkDB recently [released version 2.0](http://rethinkdb.com/blog/2.0-release/), and we at Pusher are all very excited about how creating amazing realtime apps can be even easier. [Changefeeds](http://rethinkdb.com/docs/changefeeds/ruby/), a feature introduced by RethinkDB a few versions ago, allows your system to listen to changes in your database. With this new version, this has significantly improved and opens up interesting possibilities for realtime applications.

While RethinkDB covers listening to events on your server, there is still the issue of publishing these changes to your client, with which you can build anything from news feeds to data visualizations. 

This is where a hosted message broker such as [Pusher](https://pusher.com/) comes in: by exposing RethinkDB changefeed changes as Pusher events, you can quickly achieve scalable last-mile delivery that instantly pushes database updates to the client. Not only that, but Pusher’s evented publish-subscribe approach fits the logic of real-time applications; **channels** identify the data, whether that's a table in a database or, in the case of RethinkDB, a changefeed. **Events** represent what's happening to the data: new data available, existing data being updated or data being deleted. As an added bonus, real-time features can be rolled into production fast, in the knowledge that Pusher will scale to millions of concurrent devices and connections, removing the pain of managing your own real-time infrastructure. 

To show you how this can be done, this post will guide you through how to create the type of activity streams found on the [RethinkDB website](http://rethinkdb.com). By creating a small Sinatra app, we'll quickly build the JSON feed and high-scores list you can see [in our demo](http://pusher-rethinkdb.herokuapp.com). Note that while we are using Ruby and Sinatra, one of the great things about RethinkDB’s adapters is how similar they are across all languages - so what we do here can easily be applied to the stack of your choice.

You can play with the demo [here](http://pusher-rethinkdb.herokuapp.com). If you get stuck at any point, feel free to [check out the source code](https://github.com/pusher/snake-rethinkdb/tree/322219a9593d871b1b863d45a2871d4686217074).


## Step 1: Setting Up

Firstly, if you don't already have one, you can [sign up for a free Pusher account](http://pusher.com/signup). Do keep your application credentials to hand.

If you do not have RethinkDB installed, you can install it on a Mac using Homebrew:

    $ brew install rethinkdb
  
Installation for other operating systems can be found [in their documentation](http://rethinkdb.com/docs/install/). 

To start your RethinkDB server, type into your terminal:

    $ rethinkdb
  
Now browse to <http://localhost:8080/#tables>, which leads you to a Web UI where you can create your database. Create a database called `game`, with a table called `players`. 

<div style="text-align:center">
<img alt="Create database" src="http://blog.pusher.com/wp-content/uploads/2015/05/rethinkdbgif.gif" />
</div>

In your project directory, add the Pusher and RethinkDB gems to your Gemfile, along with Sinatra for our web application.

```ruby
gem 'pusher'
gem 'rethinkdb'
gem 'sinatra'
```

Bundle install your gems, and create `app.rb` for our route-handlers, and `rethinkdb.rb` for our database configuration.

In `rethinkdb.rb`, let's connect to the database. We can also setup the Pusher instance we'll need for later using our app credentials. You can get these on [your dashboard](http://app.pusher.com). 

```ruby
require 'rethinkdb'
include RethinkDB::Shortcuts

require 'pusher'

pusher = Pusher::Client.new({
  app_id: ‘YOUR_APP_ID’,
  key: ‘YOUR_APP_KEY’,
  secret: ‘YOUR_APP_SECRET’
})


$conn = r.connect(
  host: "localhost",
  port: 28015, # the default RethinkDB port
  db: 'game',
)
```
In `app.rb`, let's just setup the bare-bones of a Sinatra application:

```ruby
require 'sinatra'
require './rethinkdb'

get '/' do 
  erb :index
end
```

## Step 2: Creating Players

As you can see in [the demo](http://pusher-rethinkdb.herokuapp.com/), whenever a player enters their name, a game of Snake starts. In the meantime, we want to create a player instance from the name the user has provided. 

This demonstration will not go heavily into the HTML and jQuery behind the app, as the snake game is based on borrowed code (that you can read up on [here](http://thecodeplayer.com/walkthrough/html5-game-tutorial-make-a-snake-game-using-html5-canvas-jquery)) and the rest is straightforward and will detract from the purpose of the tutorial: last mile delivery in a few lines of code. But if you want to dig into it, feel free to check out [the source code](https://github.com/pusher/snake-rethinkdb/tree/322219a9593d871b1b863d45a2871d4686217074).

In the case of creating users, we'll just want to send an AJAX POST to `/players` with `{name: "the user's name"}` to our Sinatra server. At this endpoint, we just want to run a simple [`insert`](http://rethinkdb.com/api/ruby/#insert) query into the `players` table:

```ruby
post '/players' do 
  name = params[:name]
  r.table("players").insert(name: name).run($conn) # pass in the connection to `run`
  "Player created!"
end
```

And it's as simple as that! If you run the app and browse to <http://localhost:8080/#dataexplorer>, running `r.table("game").table("players")`, you should see your brand-new player document. 

While this allows us to successfully create a new player, we'll probably want our server to remember them for subsequent requests, such as for submitting scores. We can just amend this endpoint to store the player's ID in a session. Conveniently, a RethinkDB query response returns the instance's ID in a `"generated_keys"` field.

```ruby
post '/players' do 
  name = params[:name]
  response = r.table("players").insert(name: name).run($conn)
  session[:id] = response["generated_keys"][0]
  "Player created!" 
end
```

## Step 3: Submitting Players' Scores

For the purpose of making the demo more fun and interactive  I've added two jQuery events to the snake code. One to [trigger the start of the game once the player has been created](https://github.com/pusher/the-snake/blob/ff400c9e59c114b654145c9b638cb63b5eb508b4/public/app.js#L39) and a second to [listen for the end of the game and retrieve the user's score](https://github.com/pusher/the-snake/blob/ff400c9e59c114b654145c9b638cb63b5eb508b4/public/app.js#L58-L60).

When the game ends, we get a score passed to the jQuery event listener. With this we just make a simple POST to `/players/score` with the params `{score: score}`. At our endpoint, let's get the player by their session ID, and update their score and high-score accordingly:

```ruby 
post '/players/score' do
  id = session[:id]
  score = params[:score]

  player = r.table("players").get(id).run($conn) # get the player

  score_update = {score: score} # our update parameters
  
  if !player["score"] || score > player["high_score"]   
    # if the player doesn't have a score yet 
    # or if the score is higher than their highest score
    score_update[:high_score] = score 
    # add the high-score to the query
  end

  r.table("player").get(id).update(score_update).run($conn) # e.g. .update(score: 94, high_score: 94)
  {success:200}.to_json
end
```

Now that we have a `high_score` key for `players` in our database, we can start rendering a static view of the leaderboard in our app, before we make it realtime. In `rethinkdb.rb`, let's build our leaderboard query.

```ruby
LEADERBOARD = r.table("players").order_by({index: r.desc("high_score")}).limit(5)
```

In order for this to work, make sure you have created an index called `"high_score"` through which to order `players`. You can do this in [your RethinkDB data explorer](http://localhost:8080/#dataexplorer) by running `r.db("game").table("players").indexCreate("high_score")`.

If on our client we wish to make a GET request to `"/leaderboard"` so that we can render leaders to the DOM, we can create that endpoint as follows:

```ruby
get '/leaderboard' do
  leaders = LEADERBOARD.run($conn)
  leaders.to_a.to_json
end
```

Using jQuery or your preferred Javascript framework, we can show a static list of the players with the highest scores:

```javascript
$.get('/leaderboard', function(leaders){
  showLeaderboard(leaders); // showLeaderboard can render leaders in the DOM.
})
``` 

## Step 4: Make Your Database Realtime

As you can see from the [demo](http://pusher-rethinkdb.herokuapp.com/), we have two realtime streams involved: a raw JSON feed of live scores, and a live leaderboard. In order to start listening to these changes, we use the RethinkDB gem's adapter for EventMachine. So add the top of `rethinkdb.rb`, add `require 'eventmachine'`. Sinatra lists EventMachine as a dependency, so it should already be available within the context of your bundle.

Seeing as we've already built our `LEADERBOARD` query above, let's dive right into how we can listen for changes regarding that query. All that is necessary is to call the `changes` method on the query, and instead of calling `run`, call `em_run`. Once we have the change, all we need is one line of Pusher code to trigger the event to the client.

```ruby
EventMachine.next_tick do # usually `run` would be sufficient - `next_tick` is to avoid clashes with Sinatra's EM loop
  LEADERBOARD.changes.em_run($conn) do |err, change|
    updated_player = change["new_val"]
   pusher.trigger(“scores”, "new_high_score", update_player)
  end
end
```
An awesome thing about RethinkDB's changefeed is that it passes the delta whenever there is a change concerning a query, so you get the old value and the new value. An example of this query, showing the previous and updated instance of the player who has achieved a high score, would be as follows:

```javascript
{
  "new_val": {
    "high_score": 6 ,
    "id":  "476e4332-68f1-4ae9-b71f-05071b9560a3" ,
    "name":  "thibaut courtois" ,
    "score": 6
  },
  "old_val": {
    "high_score": 2 ,
    "id":  "476e4332-68f1-4ae9-b71f-05071b9560a3" ,
    "name":  "thibaut courtois" ,
    "score": 1
  }
}
```

In this instance we just take the `new_val` of the change - that is, the most recently achieved high-score - and trigger a Pusher event called `"new_high_score"` on a channel called `"scores"` with that update. You can test that it works by changing a player's high score, either on your app or your RethinkDB data explorer, and heading to your debug console on <http://app.pusher.com> to view the newly-created event.

The raw JSON feed of scores also shown in our demo is also simple to implement. Let's just build the query and place it in the same EventMachine block.

```ruby
LIVE_SCORES = r.table("players").has_fields("score") 
# all the players for whom `score` isn't `nil`

EventMachine.next_tick do 
  
  ...
  
  LIVE_SCORES.changes.em_run($conn) do |err, change|
    updated_player = change["new_val"]
    pusher.trigger(“scores”, "new_score", updated_player)
  end
end
``` 

And now, in your debug console, whenever somebody's score has changed, you should see an event like this:

```javascript
{
  "high_score": 1,
  "id": "97925e44-3e8f-49cd-a34c-90f023a3a8f7",
  "name": "nacer chadli",
  "score": 1
}
```

We could put something in here where we make CURL requests to our endpoints and show the results in the Pusher Debug Console. Purpose - to show off our tooling.

## Step 5: From DB To DOM

Now that we have Pusher events firing whenever the value of our queries change, we can bind to these events on the client and [mutate the DOM](https://github.com/pusher/snake-rethinkdb/blob/322219a9593d871b1b863d45a2871d4686217074/public/app.js#L8-L20) accordingly. We simply create a new Pusher instance with our app key, subscribe to the `scores` channel, and bind callbacks to our events on that channel:

```javascript
var pusher = new Pusher("YOUR_APP_KEY");

var channel = pusher.subscribe("scores");

channel.bind("new_score", function(player){
  // append `player` to the live JSON feed of scores
});

channel.bind("new_high_score", function(player){
  // append `player` to the leaderboard
});
```

And there you have it: a simple and efficient way to update clients in realtime whenever a change happens in your database!

## Going Forward

Hopefully we've given you an insight into RethinkDB's nice querying language and awesome realtime capabilities, and how Pusher can easily be integrated to work as 'last-mile delivery' from changes in your database to your client. Aside from setting up the app itself, there is not much code involved in getting changefeeds up and running. 

The demo I have shown you is a fairly small and straightforward, however in large, production apps is possibly where the benefits of integrating Pusher with RethinkDB are greater felt. RethinkDB allows you to easily scale your database, and Pusher handles the scalability of your realtime messaging for you. Combining the two can allow developers to build powerful and reliable applications that scale.