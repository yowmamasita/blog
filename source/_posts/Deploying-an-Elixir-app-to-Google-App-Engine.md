---
title: Deploying an Elixir app to Google App Engine
date: 2016-07-19 15:04:29
tags:
---

I've been hearing a lot of good things about [Elixir](http://elixir-lang.org/) lately so I thought of trying it out and making it work with [Google App Engine](https://cloud.google.com/appengine/docs/flexible/). This post has nothing about these good things though; I've been reading and practicing a lot of functional programming lately and I recommend everyone to check it out.

To start, you need to install Elixir. Instructions are described [here](http://elixir-lang.org/install.html) to set it up. I am running Ubuntu 14.04 so I had to do run these commands:

```
wget https://packages.erlang-solutions.com/erlang-solutions_1.0_all.deb && sudo dpkg -i erlang-solutions_1.0_all.deb
sudo apt-get update
sudo apt-get install esl-erlang
sudo apt-get install elixir
```

Next, I am going to use [Phoenix](http://www.phoenixframework.org/) to create a basic web app. To use it, you need to install it as well; instructions are [here](http://www.phoenixframework.org/docs/installation).

```
mix local.hex # install Hex package manager
mix archive.install https://github.com/phoenixframework/archives/raw/master/phoenix_new.ez
```

Once done, we can start creating Phoenix projects. For our purpose, let's just create a minimal setup.

`mix phoenix.new gaetest --no-brunch --no-ecto`

This command will setup a Phoenix project in **gaetest**. You can run `mix phoenix.server` inside that directory to run your app.

We have to make some config updates to our Phoenix project so we can run it inside GAE. First, add [exrm](https://github.com/bitwalker/exrm) as a dependency. Open `mix.exs` and add this:

```
defp deps do
  [{:phoenix, "~> 1.2.0"},
   ...
   
   {:exrm, "~> 1.0.8"}]
end
```

Next, we configure our production release to run as a server in port 8080. To do this, open `config/prod.exs` and change it like this:

```
config :gaetest, Gaetest.Endpoint,
  http: [port: 8080],
  url: [host: "example.com"],
  cache_static_manifest: "priv/static/manifest.json",
  server: true
```

Now, we need to build a release. Run these commands to do that:

```
mix do deps.get, compile
MIX_ENV=prod mix phoenix.digest
MIX_ENV=prod mix compile
MIX_ENV=prod mix release
```

Next time, all we need to do is run `MIX_ENV=prod mix do phoenix.digest, compile, release` to make a new release.

Then, onto our GAE setup, we'll be using a [custom runtime](https://cloud.google.com/appengine/docs/flexible/custom-runtimes/) as Elixir is not natively supported. We need to install [Cloud SDK](https://cloud.google.com/sdk/) and [Docker](https://www.docker.com/) for this. Run `gcloud init` if this is your first time installing Cloud SDK. To make sure everything is ready, run [*gcloud preview app --help*](https://cloud.google.com/sdk/gcloud/reference/preview/app/) and *docker -v*.

Inside the **gaetest** directory, create *app.yaml* to configure our project to use a custom runtime:

```
# app.yaml

runtime: custom
vm: true
```

Next, we'll create a *Dockerfile* to describe how to run our app inside GAE:

```
# Dockerfile

FROM trenpixster/elixir

ENV VERSION 0.0.1
ENV APP_HOST 8080
EXPOSE $APP_HOST

RUN mkdir /app
WORKDIR /app
COPY ./rel/gaetest/releases/$VERSION/gaetest.tar.gz /app/gaetest.tar.gz
RUN tar -zxvf gaetest.tar.gz

WORKDIR /app/releases/$VERSION
ENTRYPOINT ["./gaetest.sh"]
CMD ["foreground"]
```

You can use other base Docker images that provide Elixir runtime, I chose [trenpixster/elixir](https://hub.docker.com/r/trenpixster/elixir/) for this example but it should run just the same. Notice that I also use the app's version number as a directory namespace. You can mount a Docker volume in `/app` so you can run different versions easily - but that's outside GAE context though.

Last step is deploying to GAE. If you haven't created a GCP project yet, you need to [create one](https://console.cloud.google.com/) and get the project ID. *You need to have billing enabled in your GCP project to use Flexible Environments.* Now run `gcloud preview app deploy --project=PROJECTID --version=VERSION app.yaml` to deploy your Elixir app. `VERSION` is just an arbitrary string. At the end of everything, your app should be available at `http://projectid.appspot.com`. Enjoy!