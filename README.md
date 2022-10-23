# Deploying a Rails API to Render

## Learning Goals

- Set up your local environment for deploying with Render
- Deploy a basic Rails application to Render

## Introduction

In this lesson, we'll be deploying a basic, standalone Rails API application to
Render. We'll give instructions to generate the application from scratch and
talk through the steps to get the code running on a Render server.

In coming lessons, we'll learn how to add more complexity to the application
with a React frontend. Since the setup for a Rails-React application is a bit
trickier, it'll be beneficial to see the setup for Rails alone first. Let's get
started!

## Environment Setup

To make sure you're able to deploy your application, you'll need to do the
following:

### Sign Up for a Render Account

You can sign up at for a free account at
[https://dashboard.render.com/register][Render signup]. Sign up using whichever
method you prefer then, from the Render dashboard, go ahead and connect Render
to your GitHub account. Once you've done that, you should see all your repos
listed in the "Connect a repository" window.

### Install the Latest Ruby Version

Verify which version of Ruby you're running by entering this in the terminal:

```console
$ ruby -v
```

We recommend version 2.7.4. If you need to upgrade you can install it using rvm:

```console
$ rvm install 2.7.4 --default
```

You should also install the latest versions of `bundler` and `rails`:

```console
$ gem install bundler
$ gem install rails
```

### Install PostgreSQL

Render requires that you use PostgreSQL for your database instead of SQLite.
PostgreSQL (or just Postgres for short) is an advanced database management
system with more features than SQLite. If you don't already have it installed,
you'll need to set it up.

#### PostgreSQL Installation for WSL

To install Postgres for WSL, run the following commands from your Ubuntu
terminal:

```console
$ sudo apt update
$ sudo apt install postgresql postgresql-contrib libpq-dev
```

Then confirm that Postgres was installed successfully:

```console
$ psql --version
```

Run this command to start the Postgres service:

```console
$ sudo service postgresql start
```

Finally, you'll also need to create a database user so that you are able to
connect to the database from Rails. First, check what your operating system
username is:

```console
$ whoami
```

If your username is "ian", for example, you'd need to create a Postgres user
with that same name. To do so, run this command to open the Postgres CLI:

```console
$ sudo -u postgres -i
```

From the Postgres CLI, run this command (replacing "ian" with your username):

```console
$ createuser -sr ian
```

Then enter `control + d` or type `logout` to exit.

[This guide][postgresql wsl] has more info on setting up Postgres on WSL if you
get stuck.

#### PostgreSQL Installation for OSX

To install Postgres for OSX, you can use Homebrew:

```console
$ brew install postgresql
```

Once Postgres has been installed, run this command to start the Postgres
service:

```console
$ brew services start postgresql
```

Phew! With that out of the way, let's get started on building our Rails
application and deploying it to Render.

## Creating a Rails App to Deploy

We'll be following the steps in the [Getting Started with Ruby on Rails on
Render][getting started with rails] guide, so if you get stuck and are looking
for more assistance, check that guide first.

The first thing we'll need to do is create our new Rails application. Make
sure you're in a non-lab directory, then run:

```console
$ rails new bird-app --api --minimal --database=postgresql
```

This will set up our app to run in API mode, with the minimum dependencies
needed, and with PostgreSQL as the database.

Next, we'll need to configure our `Gemfile.lock` file to support the same OS as
Render, which runs Ubuntu. This way, regardless of what OS you're using in
development, `bundler` will be able to install the same gems on Render using any
Ubuntu-specific gem dependencies.

`cd` into the app, and run this command:

```console
$ bundle lock --add-platform x86_64-linux
```

This will add additional platforms to your `Gemfile.lock` file that will allow
the necessary dependencies to be installed after you deploy your app.

## Building the Demo App

Next, let's set up up a migration, model, route, and controller so we have some
data to display in our application:

```console
$ rails g resource Bird name species
```

Add this data to the `db/seeds.rb` file:

```rb
Bird.create!(name: 'Black-Capped Chickadee', species: 'Poecile Atricapillus')
Bird.create!(name: 'Grackle', species: 'Quiscalus Quiscula')
Bird.create!(name: 'Common Starling', species: 'Sturnus Vulgaris')
Bird.create!(name: 'Mourning Dove', species: 'Zenaida Macroura')
```

Then run this command to generate the database and run the migrations and seed
file:

```console
$ rails db:create db:migrate db:seed
```

> `rails db:create` creates a new PostgreSQL database to be associated with your
> application based on the configuration in the `config/database.yml` file.
> Unlike with SQLite, the actual database file isn't created in the `db` folder;
> it lives elsewhere in your file system, depending on your PostgreSQL
> configuration. If you have problems with this step, see the
> **Troubleshooting** section below.

Next, edit the `app/birds_controller.rb` file and add an `index` action:

```rb
  # GET /birds
  def index
    birds = Bird.all
    render json: birds
  end
```

Finally, open `config/routes.rb`, un-comment out the root path definition and
update it to:

```rb
root "birds#index"
```

To make sure the app works locally before deploying, run `rails s`. If you visit
either [http://localhost:3000](http://localhost:3000) or
[http://localhost:3000/birds](http://localhost:3000/birds), you should see the
JSON for the list of birds.

## Preparing your App for Deployment

Before we can deploy our app, we need to make a few modifications.

First, open the `config/database.yml` file, scroll down to the `production`
section, and update the code to the following:

```yml
production:
  <<: *default
  url: <%= ENV['DATABASE_URL'] %>
```

Next, open `config/puma.rb` and find the section shown below. Here, you will
un-comment out two lines of code and make one small edit:

```rb
# Specifies the number of `workers` to boot in clustered mode.
# Workers are forked web server processes. If using threads and workers together
# the concurrency of the application would be max `threads` * `workers`.
# Workers do not work on JRuby or Windows (both of which do not support
# processes).
#
workers ENV.fetch("WEB_CONCURRENCY") { 4 } ### CHANGE: Un-comment out this line; update the value to 4

# Use the `preload_app!` method when specifying a `workers` number.
# This directive tells Puma to first boot the application and load code
# before forking the application. This takes advantage of Copy On Write
# process behavior so workers use less memory.
#
preload_app! ### CHANGE: Un-comment out this line
```

Next, open the `config/environments/production.rb` file and find the following
line:

```rb
config.public_file_server.enabled = ENV["RAILS_SERVE_STATIC_FILES"].present?
```

Update it to the following:

```rb
config.public_file_server.enabled = ENV['RAILS_SERVE_STATIC_FILES'].present? || ENV['RENDER'].present?
```

Finally, inside the `bin` folder create a `birds-build.sh` script and copy the
following into it:

```sh
#!/usr/bin/env bash
# exit on error
set -o errexit

bundle install
# bundle exec rake assets:precompile # These lines are commented out because we have an API only app
# bundle exec rake assets:clean
bundle exec rake db:migrate 
bundle exec rake db:seed
```

Our API-only app doesn't include any assets, so we've commented out the lines to
precompile and clean them.

Then run the following command in the terminal to update the permissions on the
script and make sure it's executable:

```console
chmod a+x bin/birds-build.sh
```

There will be no output from this command, but if you run `ls -l bin`, you
should see the three x's in the permissions string, indicating that the file is
executable:

```sh
-rwxr-xr-x  1 <username>  staff   253 Oct 23 07:44 birds-build.sh
```

## Deploying

### Push the Code to GitHub

In order to deploy our app to Render, we first need to create a remote repo and
push our code up. Start by making a commit to save your local changes:

```console
$ git add .
$ git commit -m 'Initial commit'
```

Then, on the repository list page of your GitHub account, click the green "New"
button in the upper right corner. (Alternatively, you can navigate to
[https://github.com/new](https://github.com/new)). In the form that opens, enter
a name for your repo (`bird-app` makes sense) and make sure "Public" is
selected. You can leave everything else as is. Click the "Create repository"
button at the bottom of the page.

On the next page, copy the code in the "push an existing repository from the
command line" section and run it in your terminal:

```console
git remote add origin git@github.com:<your-github-name>/bird-app.git
git branch -M main
git push -u origin main
```

When you refresh the GitHub page, you should see that your code has been pushed
up.

### Create the Database on Render

One limitation of Render is that it only allows one PostgreSQL instance to be
created per user account. With this instance, we can create an app, give it some
seed data, and deploy it, storing the data in the PostgreSQL instance's
database. But then what happens if you want to deploy additional apps to Render?
You can probably see how using a single database for multiple apps could get
complicated very quickly and potentially cause problems. Fortunately, Render
allows users to create [multiple databases within a single PostgreSQL
instance][multiple dbs] so you can have separate databases for each app you
deploy.

Let's start by creating the PostgreSQL instance.

Go to the [Render dashboard][], click the "New +" button and select
"PostgreSQL". Enter a name for your database — this can be whatever you like.
We're using `my_database`. The remaining fields can be left as is.

![Creating a new database](https://curriculum-content.s3.amazonaws.com/phase-4/deploying-rails-api/create-database.png)

Scroll to the bottom of the page and click "Create Database". Leave the database
page open — you'll need to copy information from it as we proceed.

Next, let's create a database specifically for our bird app. We'll do this using
the PostgreSQL interactive terminal, [`psql`][psql].

The command to launch the interactive terminal is provided in the Render
database page. Scroll down to the "Connections" section and copy the PSQL
command. Paste it into your terminal and press enter. This command connects you
to the remote database. You should now see the `psql` command prompt,
`my_database=>`, and if you run the `\l` command to list the databases, you'll
see a table that includes `my_database`.

To create the database for our bird app, we'll run the `CREATE DATABASE` SQL
command. Again, you can name your database whatever you like; we're using
`bird_app_db`. Be sure to include the semi-colon at the end of the command:

```sql
CREATE DATABASE bird_app_db;
```

Now if you run the `\l` command again, you should see that `bird_app_db` has
been added to the list of databases.

You can now exit `psql` using the `\q` command.

**Note**: The Render database page will not show the information about the
`bird_app_db` database; it will only show the name you assigned when you created
the PostgreSQL instance on Render (`my_database`). To see any other databases
you have on your PostgreSQL instance, you'll need to use `psql`. For now, be
sure to make a note of your new database's name as we'll need to use it in the
next step.

### Create the Web Service on Render

Open a new browser tab and navigate back to the Render dashboard. Click the "New
+" button and select "Web Service". You'll see a list of all the repositories in
your GitHub account. Find the repo you just created for the bird app and click
the "Select" button.

In the page that opens, enter a name for your app and make sure the Environment
is set to Ruby:

![Creating a new web service](https://curriculum-content.s3.amazonaws.com/phase-4/deploying-rails-api/create-web-service.png)

Scroll down and set the Build Command to `./bin/birds-build.sh` and the Start
Command to `bundle exec puma -C config/puma.rb`:

![Update Build and Start commands](https://curriculum-content.s3.amazonaws.com/phase-4/deploying-rails-api/update-commands.png)

Next, scroll down and click the "Advanced" button, then click "Add Environment
Variable." Enter `DATABASE_URL` as the key, then navigate back to the tab you
left open with your database information. Click the "Connect" button in the
upper right corner, copy the Internal Database URL, and paste it into the value
box. You will see a long URL that ends with `my_database`. You'll need to remove
the `my_database` and replace it with `bird_app_db` (or whatever you named it).
It should look something like this:

```sh
postgres://my_database_user:#################################################/bird_app_db
```

Click "Add Environment Variable" again. Add `RAILS_MASTER_KEY` as the key. The
value is in the `config/master.key` file in your app's files. Copy the value and
paste it in.

![Add environment variables](https://curriculum-content.s3.amazonaws.com/phase-4/deploying-rails-api/add-env-variables.png)

Scroll down to the bottom of the page and click "Create Web Service". The deploy
process will begin automatically. Warning: this process can take a while! You
might want to go for a walk or get a snack.

When the deploy process is complete, you should see something like this in the log:

![Log showing successful build and deploy](https://curriculum-content.s3.amazonaws.com/phase-4/deploying-rails-api/successful-deploy-log.png)

Click on your app's URL in the upper left corner of the screen (just below the
name of the app). Once the page has loaded (which may take a few moments), you
should see the JSON for the list of birds. If you get a "Page not found" error,
wait a few minutes and refresh the page.

## Adding New Features

Since Render integrates the deploying process with GitHub, it's straightforward
to add new features to your code and deploy them. Let's start by adding a new
controller action in the `BirdsController`:

```rb
def show
  bird = Bird.find(params[:id])
  render json: bird
rescue ActiveRecord::RecordNotFound
  render json: "Bird not found", status: :not_found
end
```

Test your code locally by running `rails s` and visiting
[http://localhost:3000/birds/1](http://localhost:3000/birds/1).

Next, we'll commit and push our changes and deploy the app. Before we do that,
however, we need to modify the build script, `bin/birds-build.sh`. The reason
for this is the script currently contains the `db:seed` command. If we keep that
command in the script, it will re-seed the data every time we push up a change,
resulting in duplicate records. Go ahead and open the script and delete (or
comment out) the last line. (Another option is to leave the script as is and
delete the seed data from the `db/seeds.rb` file.)

Now we can go ahead and make a commit and push it to GitHub:

```console
$ git add app/controllers/birds_controller.rb
$ git commit -m 'Add show action'
$ git push
```

After pushing the new code, start the deploy process by returning to the app's
page on Render, clicking the "Manual Deploy" button in the upper right corner
and selecting "Deploy latest commit." You will now see the deploy based on the
new commit in progress:

![Deploying new action](https://curriculum-content.s3.amazonaws.com/phase-4/deploying-rails-api/deploying-new-content.png)

Once the deploy is complete, refresh the page and verify that the show action is
working. Remember that it may take a few minutes for the new content to become
available.

## Troubleshooting

If you ran into any errors along the way, here are some things you can try to
troubleshoot:

- If you're on a Mac and got a server connection error when you tried to run
  `rails db:create`, one option for solving this problem for Mac users is to
  install the Postgres app. To do this, first uninstall `postgresql` by running
  `brew remove postgresql`. Next, download the app from the
  [Postgres downloads page][] and install it. Launch the app and click
  "Initialize" to create a new server. You should now be able to run
  `rails db:create`.

- If you're using WSL and got the following error running `rails db:create`:

  ```txt
  PG::ConnectionBad: FATAL:  role "yourusername" does not exist
  ```

  The issue is that you did not create a role in Postgres for the default user
  account. Check [this video](https://www.youtube.com/watch?v=bQC5izDzOgE) for
  one possible fix.

- If your app failed to deploy at the build stage, make sure your local
  environment is set up correctly by following the steps at the beginning of
  this lesson. Check that you have the latest versions of Ruby and Bundler, and
  ensure that PostgreSQL was installed successfully.

- If you deployed successfully, but you ran into issues when you visited the
  site, make sure you migrated and seeded the database. Also, make sure that
  your application works locally and try to debug any issues on your local
  machine before re-deploying. You can also check the deployment log on the
  app's page in the Render dashboard.

## Notes on Render's Free Tier

### Free Web Services

According to Render's documentation on its [Free Web
Services](https://render.com/docs/free#free-web-services):

> Web Services on the free plan are automatically spun down after 15 minutes of
> inactivity. When a new request for a free service comes in, Render spins it up
> again so it can process the request.

> This can cause a response delay of up to 30 seconds for the first request that
> comes in after a period of inactivity.

This means that, when you try to navigate to your app in the browser, it might
take a while to load — in our testing, it was often longer than 30 seconds, and
sometimes quite a bit longer!

### Free PostgreSQL Databases

With Render's free tier, databases expire after 90 days. This means that, before
the end of the 90 days, you will need to back up your databases, delete the
PostgreSQL instance from Render, create a new PostgreSQL instance, and populate
it from the database backups. Render should send an email warning you that your
database will be expiring soon.

## Backing Up and Recreating Your Databases on Render

**Note:** You do not need to complete the steps in this section until you get
close to the end of the 90 days. Feel free to skip this section until then if
you prefer.

Before we launch into the instructions for backing up and recreating your Render
PostgreSQL instance, let's take a closer look at the PSQL connection string. To
get the connection string, navigate to your database in the Render dashboard,
click the connect button then the "External Connection" tab, then copy/paste the
"PSQL Command" into your text editor. It will look something like this:

```text
PGPASSWORD=############# psql -h ################-postgres.render.com -U my_database_user my_database
```

The first element is the password for your database, which will be a
32-character string. Next is the command you're running, in this case, `psql`.
The next component is the host (indicated by the `-h` flag), which will end with
"-postgres.render.com". Next is the name of the database user (indicated by the
`-U` flag), followed by the name of the database itself.

If you paste the connection string into your terminal and run it, then run the
`\l` command to list your databases, you'll see that the username and database
name match the entry for your PostgreSQL instance in the table that's printed.
You should also see the `bird_app_db` in the list. Note that the username is the
same as for the PostgreSQL instance.

### Backing Up Our Databases

To back up our databases, we're going to modify the connection string. We'll do
that once for each of the two databases we need to back up. We recommending
making the edits to the string in your text editor then copy/pasting it into the
terminal when you're done.

> **Note:** Technically, we don't need to back up `my_database` because we don't
> have any data stored in it. We'll do it here to demonstrate the process of
> backing up and restoring multiple databases.

The first part of the string, the password, will remain the same. The `psql`
command should be updated to `pg_dump` instead. The host and username should
also stay the same. After that, we'll add the following options:

```sh
--format=custom --no-acl --no-owner
```

Then finally comes the name of the database we're backing up, followed by a `>`,
then the name of the file that we'll store the backup in, with the `.sql`
extension:

```sh
my_database > my_database.sql
```

The updated string will look something like this:

```sh
PGPASSWORD=############# pg_dump -h ################-postgres.render.com -U my_database_user --format=custom --no-acl --no-owner my_database > my_database.sql
```

Go ahead and run the command. It will not print any output, but if you run `ls`,
you should see the newly-created `.sql` file in the current directory.

Repeat the process above for the `bird_app_db` database. The string will be
exactly the same except for the last two elements: the name of the database and
the name of the backup file.

### Replacing the Expiring PostgreSQL Instance

Return to the database page in the Render dashboard. Scroll to the bottom of the
page, click "Delete Database" and follow the instructions. Next, create a new
PostgreSQL instance by clicking the "New +" button and selecting PostgreSQL.
Give your new instance a name (we'll use `my_new_db`), then scroll down to the
bottom of the page, and click "Create Database."

### Restoring the Databases to the New Instance

Once the new instance has been created (which might take a few minutes), copy
the PSQL connection string and paste it into your code editor. Once again, we'll
be editing this string, this time to create the command to restore the databases
from the backup files.

The restore string will consist of the following:

- the database password,
- the `pg_restore` command,
- the options: `--verbose --clean --no-acl --no-owner`,
- the host,
- the `-d` flag (for `dbname`) followed by the name of the new database you're
  restoring the data to (`my_new_db`),
- the name of the `.sql` file you're restoring from

The final string will look something like this:

```text
PGPASSWORD=################ pg_restore --verbose --clean --no-acl --no-owner -h #################-postgres.render.com -U my_new_db_user -d my_new_db my_database.sql
```

Run the command in the terminal. You should see a flurry of activity as it
creates the database tables.

We've restored the main database for our PostgreSQL instance, so the next step
is to restore the database we created for our bird app. Before you can do that,
you'll first need to create the new database in PSQL. Execute the PSQL
connection string for the new PostgreSQL instance to launch the interactive
terminal, then run the CREATE DATABASE command (you can use the same database
name or a different one, as you prefer):

```sql
CREATE DATABASE bird_app_db;
```

Once you've done that, you can restore the bird app data to the new database.
Update the restore command, changing the database name and backup file name, and
run it in the terminal.

### Connecting the New Databases to Your Web Services

The final step in the process is to update the bird app's Web Service so that it
points to the new database. From the Render database, select the bird app, then
click "Environment" in the nav on the left. Delete the value associated with the
DATABASE_URL key and replace it with the Internal URL for the new `bird_app_db`
database. Now, if you click the app's URL, you should see the JSON with the list
of birds.

## Conclusion

Congrats on deploying your first Rails app to the world wide web! Understanding
the deployment process and what it takes to run your application on another
computer is an important step toward becoming a full-stack developer. Like
anything new, this process can be daunting the first time you try it, but with
practice and exposure, you'll build confidence over time.

In the next lesson, we'll work on deploying a more complex application with a
Rails API backend and a React frontend, and talk through some of the challenges
of running these two applications together.

## Check For Understanding

Before you move on, make sure you can answer the following questions:

1. When creating a new Rails app from the terminal, what additional flag do you
   need to use to be able to deploy it on Render?
2. What familiar process is used for deploying code to Render?

## Resources

- [Getting Started with Ruby on Rails on Render][getting started with rails]
- [Render Databases Guide][databases guide]

[Render signup]: https://dashboard.render.com/register
[postgresql wsl]: https://docs.microsoft.com/en-us/windows/wsl/tutorials/wsl-database#install-postgresql
[getting started with rails]: https://render.com/docs/deploy-rails
[Render dashboard]: https://dashboard.render.com/
[databases guide]: https://render.com/docs/databases
[postgres downloads page]: https://postgresapp.com/downloads.html
[multiple dbs]: https://render.com/docs/databases#multiple-databases-in-a-single-postgresql-instance
[psql]: https://www.postgresql.org/docs/current/app-psql.html
