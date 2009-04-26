Capistrano Handbook
===================

## Index

* [Simple. The Way it Should Be](simple_the_way_it_should_be)
* [Building Blocks Variables & Tasks](building_blocks_varaibles_and_tasks)
  * [Tasks](building_blocks_varaibles_and_tasks/tasks)
  * [Variables](building_blocks_varaibles_and_tasks/variables)
* [Local Vs. Remote Tasks](/local_vs_remote_tasks)
* [Rights, Users and Privileges](/rights_users_privileges)
* [Command Line Options](/command_line_options)

### Anatomy of a Capistrano installation

Typically inside your application you will have a structure not dissimilar to the following:

    /
    |- public/
    |- config/
    |- application/

Regardless of your application's nature you would usually expect to have these directories; whether or not they exist is largely academic.

Capistrano relies on you having a `config` directory into which it will install a `deploy.rb` file that contains your settings.

The `deploy.rb` file is loaded when you call `cap` on the command line; in the event that you aren't in the root of your application (or more accurately that there isn't a `capfile`) in your present working directory, `cap` will search up the directory structure until it finds one, this may include your home directory.

**Beware of this when dealing with multiple projects, or nested projects - this feature was intended so that you could run a deploy, or open a [`cap shell`](http://weblog.jamisbuck.org/2006/9/21/introducing-the-capistrano-shell) without moving to the root of your application.**

Typically `capifying` an application will create something akin to the following:

    set :application, "set your application name here"
    set :repository,  "set your repository location here"

    # If you aren't deploying to /u/apps/#{application} on the target
    # servers (which is the default), you can specify the actual location
    # via the :deploy_to variable:
    # set :deploy_to, "/var/www/#{application}"

    # If you aren't using Subversion to manage your source code, specify
    # your SCM below:
    # set :scm, :subversion

    role :app, "your app-server here"
    role :web, "your web-server here"
    role :db,  "your db-server here", :primary => true

If your application is not separated into `application`, `web` and `database` servers, you can either set them to be the same value; or comment out, or remove the one you do not require.

Typically for a `PHP` application one would expect to comment out the `web` or `app` roles, depending on your requirements. Certain built in tasks expect to run only on one, or the other kind of server.

The `:primary => true` part of the role definitions allows you to have more than one database server, this could easily also be written either of the following two ways:

    role :db, 'db1.example.com', 'db2.example.com'
    -- or --
    role :db, 'db1.example.com', 'db2.example.com', :primary => true

If you have two servers, and neither is `primary`, or 

    role :db, 'db1.example.com', :primary => true
    role :db, 'db2.example.com'

If, for example when deploying a Rails application you only wanted `db1` to run migrations, in the first example both might.

Essentially when using the Rails deployment recipes, the `:primary` option defines where database migrations are run.

Similar attributes include `:no_release` often used for the `:web` role by some of the recipes in circulation to decide which servers should not have the code checked out to them.

Attributes like these are arbitrary and you can define some of your own, and use them to filter more precisely where your own tasks run,

You may want to read more about the [`role`](http://wiki.capify.org/index.php/Role) method as it has a few options. There is the alternate [`server`] method which works slightly differently, the examples should demonstrate how-so.

    role :db, 'www.example.com'
    role :app, 'www.example.com'
    role :web, 'www.example.com'

And the `server` method:

    server 'www.example.com', :app, :web, :db

If you have a lot of multi-function servers, or perhaps just one server running your whole application the `server` method may be a quick and easy way to remove a few LOC and a little confusion from your deployment configuration files.

Other than the shorter syntax, they are functionally equivalent.

### Building Blocks Variables & Tasks

#### Tasks

Tasks are the foundation of a Capistrano setup; collections of tasks are typically called _Recipes_.

Tasks are defined as such, and can be defined anywhere inside your `Capfile` or `deploy.rb`; or indeed any other file you care to load into the `Capfile` at runtime.

    desc "Search Remote Application Server Libraries"
    task :search_libs, :roles => :app do
      run "ls -x1 /usr/lib | grep -i xml"
    end

Lets break that down a little...

The [`desc`](http://wiki.capify.org/index.php/Desc) method defines the task description, this shows up when using `cap -T` on your application.. these are arbitrary description strings that can be used to help your users or fellow developers.

Tasks without a `desc`ription will not be listed by a default `cap -T`, but will however be listed with a `cap -Tv`. More command line options for the `cap` script will be discussed later in the handbook.

The [`task`](http://wiki.capify.org/index.php/Task) method expects a block, that is run when the task is invoked. The task can, typically contain any number of instructions, both to run locally and on your deployment target servers (`app`,`web`,`db`).

##### Namespacing Tasks 

It stands to reason that with such a wide scope of available uses, there would be potential for naming clashes, this isn't a problem localized to Capistrano; and elsewhere in the computer sciences world this has been solved with Namespacing; Capistrano is no different, take the following example:

    desc "Backup Web Server"
    task :backup_web_server do
      puts "In Example Backup Web-Server"
    end
    
    desc "Backup Database Server"
    task :backup_database_server do
      puts "In Example Backup Database-Server"
    end

Defining a task in this way, and more about how the task blocks are arranged is forthcoming; however imagine we had two tasks `backup` perhaps, that needed to work differently on different roles.. here's how namepsaces solve that problem

  namespace :web_server do
    task :backup do
      puts "In Example Backup Web-Server"
    end
  end
  
  namespace :database_server do
    task :backup do
      puts "In Example Backup Database-Server"
    end
  end

Whilst the tasks in the first example might be listed by `cap -T` as:

    backup_database_server    Backup Database Server
    backup_web_server         Backup Web Server

And invoked with either `cap backup_database_server` or `cap backup_web_server`; the second pair of examples would be listed by `cap -T` as

    database_server:backup    Backup Database Server
    web_server:backup         Backup Web Server

and similarly invoked with `cap database_server:backup` or `cap web_server:backup`

Namespaces have an implicit `default` task called if you address the namespace as if it were a task, consider the following example:

    namespace :backup do
      
      task :default do
        web
        database
      end
      
      task :web, :roles => :web do
        puts "Backing Up Web Server"
      end
      
      task :db, :roles => :db do
        puts "Backing Up DB Server"
      end
      
    end

**Note:** These are nested differently to the two previous examples, as when looked at in these terms it makes a lot more sense to namespace them this way, and simply call the following

  * To backup just the web server:

        cap backup:web

  * To backup just the db server:

        cap backup:db

  * To back up both in series:
  
        cap backup

It is important to note here that when calling tasks from within tasks, unlike with `rake` where they syntax might be something like `Rake::Tasks['backup:db'].invoke`, with Capistrano you simply name the task as if it were any other ruby method.

When calling tasks cross-namespace, or for readability you can (and often should) prefix the task call with the namespace in which the task resides, for example:

    namespace :one do
      task :default do
        test
        one.test
        two.test
      end
      task :test do
        puts "Test One Successful!"
      end
    end
    
    namespace :two do
      task :test do
        puts "Test Two Successful"
      end
    end

Calling `cap one` would output:

    Test One Successful
    Test One Successful
    Test Two Successful

This gets slightly more complicated as the namespace hierarchy becomes more intricate but the same principles always apply.

Namespaces are nestable, an example from one of the core methods is `cap deploy:web:disable` a `disable` task in the `web` namespace which in turn resides in the `deploy` namespace.

There is a `top` namespace for convenience, and it is here that methods defined outside of an explicit `namespace` block reside by default. If you are in a task inside a namespace, and you want to call a task from a _higher_ namespace, or outside of them all, prefix it with `top` and you can define the path to the task you require.

##### Existing Tasks

Unlike [`rake`](http://rake.rubyforge.org/) when you define a task that collides with the name of another (within the same namespace) it will overwrite the task rather than adding to it.

##### Chaining Tasks

As this is considered to be a feature, not a limitation of Capistrano; there is an exceptionally easy way to chain tasks, including but not limited to the `default` task for each namespace, consider the following

    namespace :deploy do
      # .. this is a default namespace with lots of its own tasks
    end
    
    namespace :notifier do
      task :email_the_boss do
        # Implement your plain ruby emailing code here with [`TMail`](http://tmail.rubyforge.org/)
      end
    end
    
    after (:deploy, "notifier:email_the_boss")

Note the different arguments, essentially it doesn't matter how you send these, strings, symbols or otherwise, they are automagically red through to ascertain what you intended, I could just have easily have written:

    after ('deploy', "notifier:email_the_boss")

The convention here would appear to be, when using a single word namespace, or task name; **pass it as a symbol** otherwise it must be a string, using the colon-separated task notation.

There are both [before](), and [after]() callbacks that you can use, and there is nothing to stop you interfering with the execution of any method that calls another, take for example that at the time of writing the implementation of `deploy:default` might look something like this:

    namespace :deploy do
      task :default
        update
        update_code
        strategy.deploy!
        finalize_update
        symlink
        restart           # <= v2.5.5
      end
    end

**More Info:** [Default Execution Path on the Wiki](http://wiki.capify.org/index.php/Default_Execution_Path)

Here we could inject a task to happen after a symlink, but before a restart by doing something like:

    after("deploy:symlink") do
      # Some more logic here perhaps
      notifier.email_the_boss
    end

Which could be simplified to:

    after("deploy:symlink", "notifier:email_the_boss")

The first example shows the shorthand anonymous-task syntax.

##### Calling Tasks

In the examples we have covered how to call tasks on the command line, how to call tasks explicitly in other tasks, and how to leverage the power of callbacks to inject logic into the default deploy strategy.

  cap production deploy
  cap staging deploy
  cap backup_database_server backup_web_server
  cap backup:database backup:web
  
##### Transactions
