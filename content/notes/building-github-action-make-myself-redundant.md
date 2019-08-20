---
layout: post
title:  Building a GitHub Action to make myself redundant
area: GitHub
---

**Note: this example is based on Actions v1, and needs to be updated because Actions v2 does things differently in a couple of places. I'll drop this note once I've updated the post.**

The details from the [previous article](/notes/github-actions-hello-world-in-ruby/) are a baseline for what I was _actually_ working on last weekend. And before we get to the fun stuff, we need some context.

## What are we making?

I help maintain the [Up for Grabs](https://up-for-grabs.net) website, which curates a list of projects that have issues open which maintainers have flagged as ideal for new contributors. It's been running for over 5 years now (omg):

```
# find the first commit associated with the current branch
$ git rev-list --max-parents=0 HEAD
b898b8a69074e4c8b39fde06f0128b6ae8d89b10

$ git show b898b8a69074e4c8b39fde06f0128b6ae8d89b10
commit b898b8a69074e4c8b39fde06f0128b6ae8d89b10
Author: Keith Dahlby <dahlbyk@gmail.com>
Date:   Wed Nov 20 14:52:10 2013 -0800

...
```

Because it's been running for so long, we've found ourselves in situations where some projects listed are no longer active. Sometimes the maintainers drop by to submit a PR to the project to update the status of the project. Sometimes they [mark the repository as archived on GitHub](https://help.github.com/en/articles/about-archiving-repositories). Sometimes the repository goes missing. But generally we don't hear from them.

Some time last year I added [a little script to the repository](https://github.com/up-for-grabs/up-for-grabs.net/pull/1044) to help with this. I wrote it in Ruby because it fitted with the rest of the project (a Jekyll static site generator), and the script scanned the projects in the repository to identify which could be cleaned up.

But this introduces a different problem - I needed to remember to run this manually, and then submit a PR to remove the projects that I found which are no longer active. I'm not sure the last time I remembered to do this, let alone submitted a pull request to clean them up.

## Goals

I had some time on Friday to finally sit down and think about porting this. These were the goals I had in mind:

 - port this script to run as a GitHub Action
 - work out a way to test the action locally, so I could iterate on it quickly
 - automate the workflow to cleanup any inactive project

## Local Testing

After scaffolding up the action in a local clone of the repository, I configured my local environment so I could hack on the script and get it working. Yes, I recommend installing Docker to do this. Trust me, it's quicker than pushing to the remote and using GitHub to test it.

I added the file to a directory named `.github/actions/cleanup-archived-projects/` and ran this command to build the Docker image:

```
$ docker build .github/actions/cleanup-archived-projects/ -t test-action
```

A tag is required here because we need to run the script to actually verify it works as expected. If the build completes without an error, we can then run the container:

```
$ docker run test-action:latest
```

This helped me to build up a rhythm with iterating on the script - make a change, build the image, run the image. The only downside with this approach is that it looks like `docker run` doesn't stream output to the terminal - it seems to buffer and emit output in chunks (or just when the container exits). There might be a way to change this behaviour, but I couldn't spot it in the documentation.

The rest of this article covers the work to port the old script to run as an Action, and how Actions influenced the end result.

### Repository access

The original script assumed it was run within a local repository, which was great because we have 750+ projects and the script needs to read the contents of each project file in the repository. That code looked like this:

```ruby
root = File.expand_path("..", __dir__) + "/"
projects = root + "_data/projects/*.yml"

results = Dir[projects].map { |f| verify_file(f) }
```

Initially I thought I'd use the API to find the projects, but it felt excessive to waste 750+ API calls when there was a perfectly good alternative available - archives. I adapted [this StackOverflow answer](https://stackoverflow.com/a/11505644/1363815) to do what I needed:

 - get the archive link for the repository using [Octokit.rb](https://github.com/octokit/octokit.rb)
 - download the archive and read it in memory
 - find all project files in the archive
 - read the contents and pass those into the `verify_file` function

```ruby
repo = ENV['GITHUB_REPOSITORY']
client = Octokit::Client.new(:access_token => ENV['GITHUB_TOKEN'])
link = client.archive_link(repo)

pattern = Regexp.new('.*\/(_data\/projects\/.*.yml)')

open(link) do |archive|
  tar_extract = Gem::Package::TarReader.new(Zlib::GzipReader.open(archive))
  tar_extract.rewind
  tar_extract.each do |entry|
    if (entry.file? && pattern.match?(entry.full_name)) then

      matches = pattern.match(entry.full_name).captures
      file_path = matches[0]
      contents = entry.read

      result = verify_file(client, file_path, contents)

      # TODO: do something with the result
    end
  end
  tar_extract.close
end
```

As this script now depends on two environment variables being set (`GITHUB_REPOSITORY` is the repository in use, `GITHUB_TOKEN` is the token to make API requests), I needed to provide those when running the Docker image. I created a Personal Access Token for my account that I provided to the Docker container like this, along with the hard-coded repository name I was going to use to run this action:

```
$ docker run -e GITHUB_TOKEN='some-value-here' -e GITHUB_REPOSITORY='shiftkey/up-for-grabs.net' test-ruby:latest
```

This pattern became very useful as I depended on more things that were part of the GitHub Actions environment. Each new environment variable needed to be provided here to ensure my code worked as expected.

It turns out I overlooked the `GITHUB_WORKSPACE` environment variable also available in the Docker container which points to a copy of the repository associated with the current commit, so I can simplify this to a more robust version of what it was doing before:

```ruby
$root_directory = ENV['GITHUB_WORKSPACE']
projects = File.join($root_directory, '_data', 'projects', '*.yml')

results = Dir.glob(projects).map { |path| verify_file(path) }
```

By the time I was done with testing the action, this was what my `docker run` setup looked like:

```
$ docker run -e GITHUB_REPOSITORY='shiftkey/up-for-grabs.net' \
   -e GITHUB_TOKEN='some-token-here' \
   -e GITHUB_SHA='$(git rev-parse origin/HEAD)' \
   -e GITHUB_WORKSPACE='/tmp/code' \
   -v $(pwd):/tmp/code \
   test-ruby:latest
```

### API access

I know from experience that a personal access token has access to 5000 API requests per hour, but I wasn't sure how requests were available to the `GITHUB_TOKEN` associated with the GitHub Action. I suspected it would be less because an early test caused the action to run two builds concurrently, and the run would emit output like this, which suggests I could easily trigger rate-limiting:

```
Project is active: '_data/projects/nikola.yml'
Project is active: '_data/projects/nlog.yml'
Project is active: '_data/projects/nmodbus4.yml'
Encountered error while trying to validate '_data/projects/nodatime.yml' - Unknown exception for file: GET https://api.github.com/repos/nodatime/nodatime: 403 - API rate limit exceeded for installation ID [id]. // See: https://developer.github.com/v3/#rate-limiting
Encountered error while trying to validate '_data/projects/node-cognitive-services.yml' - Unknown exception for file: GET https://api.github.com/repos/joshbalfour/node-cognitive-services: 403 - API rate limit exceeded for installation ID [id]. // See: https://developer.github.com/v3/#rate-limiting
Encountered error while trying to validate '_data/projects/node-gqgeiger.yml' - Unknown exception for file: GET https://api.github.com/repos/ihatecsv/node-gqgeiger: 403 - API rate limit exceeded for installation ID [id]. // See: https://developer.github.com/v3/#rate-limiting
...
```

I hadn't seen any warnings like this locally so I put together a simple function to check, which I ran before any calls I made with Octokit:

```ruby
def check_rate_limit
  rate_limit = $client.rate_limit

  remaining = rate_limit.remaining
  resets_in = rate_limit.resets_in
  limit = rate_limit.limit

  remaining_percent = (remaining * 100) / limit

  if (remaining % 10 == 0 && remaining_percent < 20) then
    puts "Rate limit: #{remaining}/#{limit} - #{resets_in}s before reset"
  end

  if (remaining == 0) then
    puts "This script is currently rate-limited by the GitHub API"
    puts "Marking as inconclusive to indicate that no further work will be done here"
    exit 78
  end
end
```

This helpfully inserted messages within the script execution to track how close I was to the actual limits:

```
Project is active: '_data/projects/Aidre.yml'
Project is active: '_data/projects/AlaSQL.yml'
Project is active: '_data/projects/AlgebraicEffects.yml'
Rate limit: 190/1000 - 3295s before reset
Project is active: '_data/projects/Ancient-Beast.yml'
Project is active: '_data/projects/AntaniXml.yml'
Project is active: '_data/projects/ApplicationInsights-dotnet.yml'
Project is active: '_data/projects/Aton.yml'
```

It showed that I have 1000 API calls to spend each hour. That suggests I could, at best, run this script every hour to iterate over the 750+ projects currently tracked by Up for Grabs without hitting the limit.

The set of projects listed here is fairly stable, with new projects being added fairly regularly. As this gets closer to 1000 I'm going to have to think about how to support being able to spread the work over multiple runs, especially if I want to do more with the GitHub API.

### Do more stuff in the script

Speaking of doing more with the GitHub API - the original script simply listed the project files it could not find using the GitHub API, and I would manually go through and remove them from Git to then make a PR. I decided I want to automate that.

This script should:

 - check for any open pull request that deletes the file associated with the project
    - if a pull request exists matching this criteria, don't bother submitting a duplicate pull request
 - if none found, open a pull request which deletes the project file
    - this makes it easy for the reviewer to say ["LGTM!"](https://www.dictionary.com/e/acronyms/lgtm/) to approve the PR and merge the pull request

Here's the function to do the first step:

```ruby
def find_pull_request_removing_file (repo, path)
  check_rate_limit
  prs = $client.pulls(repo)

  found_pr = nil

  prs.each { |pr|
    check_rate_limit
    files = $client.pull_request_files(repo, pr.number)
    found = files.select { |f| f.filename == path && f.status == 'removed' }

    if (found.length > 0) then
      found_pr = pr
      break
    end
  }

  found_pr
end
```

And here's the second step:

```ruby
def create_pull_request_removing_file (repo, path, reason)
  puts "Creating new pull request for path '#{path}'"
  file_name = File.basename(path, '.yml')
  branch_name = "projects/deprecated/#{file_name}"

  sha = ENV['GITHUB_SHA']

  short_ref = "heads/#{branch_name}"

  begin
    check_rate_limit
    foundRef = $client.ref(repo, short_ref)
  rescue
    foundRef = nil
  end

  begin
    check_rate_limit
    if (foundRef == nil) then
      puts "Creating ref for '#{short_ref}' to point to '#{sha}'"
      $client.create_ref(repo, short_ref, sha)
    else
      puts "Updating ref for '#{short_ref}' from #{foundRef.object.sha} to '#{sha}'"
      $client.update_ref(repo, short_ref, sha, true)
    end

    check_rate_limit
    content = $client.contents(repo, :path => path, :ref => 'gh-pages')

    check_rate_limit
    $client.delete_contents(repo, path, "Removing deprecated project from list", content.sha, :branch => branch_name)

    check_rate_limit
    $client.create_pull_request(repo, "gh-pages", branch_name, "Deprecated project: #{file_name}.yml", getPullRequestBody(reason))
  rescue
    puts  "Unable to create pull request to remove project #{path} - '#{$!.to_s}''"
    nil
  end
end
```

And we run this code using the results from `verify_file`:

```ruby
results.each { |result|
  file_path = result[:path]

  if (result[:deprecated]) then
    puts "Project is considered deprecated: '#{file_path}' - reason '#{result[:reason]}'"

    pr = find_pull_request_removing_file(repo, file_path)

    if pr != nil then
      puts "Project #{file_path} has existing PR ##{pr.number} to remove file..."
    else
      pr = create_pull_request_removing_file(repo, file_path, result[:reason])
      if (pr != nil) then
        puts "Opened PR ##{pr.number} to remove project '#{file_path}'..."
      end
    end
    errors += 1
  elsif (result[:error] != nil)
    puts "Encountered error while trying to validate '#{file_path}' - #{result[:error]}"
    errors += 1
  else
    if (verbose != nil) then
      puts "Active project found: '#{file_path}'"
    end
    success += 1
  end
}
```

If you know a bit of Ruby and are cringing at my code by this point, I totally understand. I've included a link to the PR at the end of this post if you'd like to suggest ways to make this more elegant or readable. I've mostly been focused on getting something working here.

## Scheduling

In the [previous article](/notes/github-actions-hello-world-in-ruby/) I configured the workflow to run on every `push` to the repository. That feels like overkill because of the rate-limiting as well as the number of projects I need to enumerate and check.

I changed the workflow file to run once every 24 hours:

```
workflow "Daily Work" {
  on = "schedule(0 18 * * *)"
  resolves = ["Cleanup archived projects"]
}

action "Cleanup archived projects" {
  uses = "./.github/actions/cleanup-archived-projects"
  secrets = ["GITHUB_TOKEN"]
}
```

This action runs at 18:00 UTC every day, which should be sufficent to stay on top of any stale projects. I'll probably scale it back later to weekly.

## Is it done?

You can find the pull request to add this action to Up for Grabs at [`up-for-grabs.net#1257`](https://github.com/up-for-grabs/up-for-grabs.net/pull/1257), as well as an example of what a pull request to remove a deprecated project looks like in [`up-for-grabs.net#1263`](https://github.com/up-for-grabs/up-for-grabs.net/pull/1263).

Future ideas that I'm thinking about:

 - Caching open pull requests and pull request files to save on API calls
 - Investigate whether I can speed up the code using concurrency or parallelization
 - Making the code more maintainable
 - Explore whether we could merge information from the GitHub API with project data when publishing the site
