# Running Migrations, etc.

A very common task as part of deployment is the ability to run migrations, or other
automated prep work prior to starting the new version. There are a number of approaches
I've seen people take using the primitives Distillery provides, however my favored approach
is one that I have not seen people use yet, and it surprises me because it is so easy and feels
much more comfortable to use.

The approach is the following:

- Define a module which will execute the migrations, this is a common
  requirement of all approaches to running migrations in a release.
- Define a custom command which will execute this module for you without
  requiring that you type the module, function, and arguments yourself.

## Migration Module

The following code is an example of a module which will run your Ecto migrations:

```elixir
defmodule MyApp.ReleaseTasks do

  @start_apps [
    :crypto,
    :ssl,
    :postgrex,
    :ecto
  ]

  def myapp, do: Application.get_application(__MODULE__)

  def repos, do: Application.get_env(myapp(), :ecto_repos, [])

  def seed do
    me = myapp()

    IO.puts "Loading #{me}.."
    # Load the code for myapp, but don't start it
    :ok = Application.load(me)

    IO.puts "Starting dependencies.."
    # Start apps necessary for executing migrations
    Enum.each(@start_apps, &Application.ensure_all_started/1)

    # Start the Repo(s) for myapp
    IO.puts "Starting repos.."
    Enum.each(repos(), &(&1.start_link(pool_size: 1)))

    # Run migrations
    migrate()

    # Run seed script
    Enum.each(repos(), &run_seeds_for/1)

    # Signal shutdown
    IO.puts "Success!"
    :init.stop()
  end

  def migrate, do: Enum.each(repos(), &run_migrations_for/1)

  def priv_dir(app), do: "#{:code.priv_dir(app)}"

  defp run_migrations_for(repo) do
    app = Keyword.get(repo.config, :otp_app)
    IO.puts "Running migrations for #{app}"
    Ecto.Migrator.run(repo, migrations_path(repo), :up, all: true)
  end

  def run_seeds_for(repo) do
    # Run the seed script if it exists
    seed_script = seeds_path(repo)
    if File.exists?(seed_script) do
      IO.puts "Running seed script.."
      Code.eval_file(seed_script)
    end
  end

  def migrations_path(repo), do: priv_path_for(repo, "migrations")

  def seeds_path(repo), do: priv_path_for(repo, "seeds.exs")

  def priv_path_for(repo, filename) do
    app = Keyword.get(repo.config, :otp_app)
    repo_underscore = repo |> Module.split |> List.last |> Macro.underscore
    Path.join([priv_dir(app), repo_underscore, filename])
  end
end
```

## Custom Command

Place the following shell script at `rel/commands/migrate.sh`:

```bash
#!/bin/sh

$RELEASE_ROOT_DIR/bin/myapp command Elixir.MyApp.ReleaseTasks seed
```

For more info on shell variables look at the [Shell Script API](https://hexdocs.pm/distillery/shell-script-api.html#environment-variables).

## Tying it all together

Now that we have our custom command and migrator module defined, we just need to set up our config appropriately:

```elixir
...

release :myapp do
  ...
  set commands: [
    "migrate": "rel/commands/migrate.sh"
  ]
end

...
```

Now, once you've deployed your application, you can run migrations with `bin/myapp migrate`. Easy as pie.

## Thoughts

There are other approaches that may make more sense for your use case, for example, automatically running migrations
by defining a pre-start hook which does basically the same thing as above, just in a hook instead of a command. You can
even define the command, and execute the command as part of the hook, giving you the flexibility of both approaches.

Custom commands give you a lot of power to express potentially complex operations as a terse statement. I would encourage
you to use them for these types of tasks rather than using the raw `rpc` and `command` constructs!
