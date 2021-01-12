# frozen_string_literal: true

require 'puppet_litmus/rake_tasks' if Bundler.rubygems.find_name('puppet_litmus').any?
require 'puppetlabs_spec_helper/rake_tasks'
require 'puppet-syntax/tasks/puppet-syntax'
require 'puppet_blacksmith/rake_tasks' if Bundler.rubygems.find_name('puppet-blacksmith').any?
require 'github_changelog_generator/task' if Bundler.rubygems.find_name('github_changelog_generator').any?
require 'puppet-strings/tasks' if Bundler.rubygems.find_name('puppet-strings').any?
require_relative 'lib/common_events_library/version.rb'

def changelog_user
  return unless Rake.application.top_level_tasks.include? "changelog"
  returnVal = nil || JSON.load(File.read('metadata.json'))['author']
  raise "unable to find the changelog_user in .sync.yml, or the author in metadata.json" if returnVal.nil?
  puts "GitHubChangelogGenerator user:#{returnVal}"
  returnVal
end

def changelog_project
  return unless Rake.application.top_level_tasks.include? "changelog"

  returnVal = nil
  returnVal ||= begin
    metadata_source = JSON.load(File.read('metadata.json'))['source']
    metadata_source_match = metadata_source && metadata_source.match(%r{.*\/([^\/]*?)(?:\.git)?\Z})

    metadata_source_match && metadata_source_match[1]
  end

  raise "unable to find the changelog_project in .sync.yml or calculate it from the source in metadata.json" if returnVal.nil?

  puts "GitHubChangelogGenerator project:#{returnVal}"
  returnVal
end

def changelog_future_release
  return unless Rake.application.top_level_tasks.include? "changelog"
  returnVal = "v%s" % JSON.load(File.read('metadata.json'))['version']
  raise "unable to find the future_release (version) in metadata.json" if returnVal.nil?
  puts "GitHubChangelogGenerator future_release:#{returnVal}"
  returnVal
end

def servicenow_params(args)
  args.names.map do |name|
    args[name] || ENV[name.to_s.upcase]
  end
end

# Executes a command locally.
#
# @param command [String] command to execute.
# @return [Object] the standard out stream.
def run_local_command(command)
  stdout, stderr, status = Open3.capture3(command)
  error_message = "Attempted to run\ncommand:'#{command}'\nstdout:#{stdout}\nstderr:#{stderr}"
  raise error_message unless status.to_i.zero?

  stdout
end


PuppetLint.configuration.send('disable_relative')

if Bundler.rubygems.find_name('github_changelog_generator').any?
  GitHubChangelogGenerator::RakeTask.new :changelog do |config|
    raise "Set CHANGELOG_GITHUB_TOKEN environment variable eg 'export CHANGELOG_GITHUB_TOKEN=valid_token_here'" if Rake.application.top_level_tasks.include? "changelog" and ENV['CHANGELOG_GITHUB_TOKEN'].nil?
    config.user = "#{changelog_user}"
    config.project = "#{changelog_project}"
    config.future_release = "#{changelog_future_release}"
    config.exclude_labels = ['maintenance']
    config.header = "# Change log\n\nAll notable changes to this project will be documented in this file. The format is based on [Keep a Changelog](http://keepachangelog.com/en/1.0.0/) and this project adheres to [Semantic Versioning](http://semver.org)."
    config.add_pr_wo_labels = true
    config.issues = false
    config.merge_prefix = "### UNCATEGORIZED PRS; GO LABEL THEM"
    config.configure_sections = {
      "Changed" => {
        "prefix" => "### Changed",
        "labels" => ["backwards-incompatible"],
      },
      "Added" => {
        "prefix" => "### Added",
        "labels" => ["feature", "enhancement"],
      },
      "Fixed" => {
        "prefix" => "### Fixed",
        "labels" => ["bugfix"],
      },
    }
  end
else
  desc 'Generate a Changelog from GitHub'
  task :changelog do
    raise <<EOM
The changelog tasks depends on unreleased features of the github_changelog_generator gem.
Please manually add it to your .sync.yml for now, and run `pdk update`:
---
Gemfile:
  optional:
    ':development':
      - gem: 'github_changelog_generator'
        git: 'https://github.com/skywinder/github-changelog-generator'
        ref: '20ee04ba1234e9e83eb2ffb5056e23d641c7a018'
        condition: "Gem::Version.new(RUBY_VERSION.dup) >= Gem::Version.new('2.2.2')"
EOM
  end
end

def get_pe_master()
  provision_hash = YAML.load_file('./inventory.yaml')
  provision_hash['groups'][1]['targets'][0]['uri']
end

def get_agent()
  provision_hash = YAML.load_file('./inventory.yaml')
  provision_hash['groups'][1]['targets'][1]['uri']
end

def module_path()
  File.join(Dir.pwd, 'spec', 'fixtures', 'modules')
end

def task_prefix(hostname)
  "bundle exec bolt task run --modulepath spec/fixtures/modules --target #{hostname}"
end

namespace :launch do
  require 'puppet_litmus/rake_tasks'
  require_relative './spec/support/acceptance/helpers'
  include TargetHelpers

  desc 'Provisions the VMs. This is currently just the master'
  task :provision_vms do
    if File.exist?('inventory.yaml')
      # Check if a master VM's already been setup
      begin
        master_uri = master.uri
        puts("A master VM at '#{master_uri}' has already been set up")
        next
      rescue TargetNotFoundError
        # Pass-thru, this means that we haven't set up the master VM
      end
    end

    Rake::Task['litmus:provision_list'].invoke('master')
    Rake::Task['litmus:provision_list'].reenable
    Rake::Task['litmus:provision_list'].invoke('agent')
  end

  desc 'Install PE'
  task :setup_pe do
    master.bolt_run_script('spec/support/acceptance/install_pe.sh')
    # Setup hiera-eyaml config
    master.bolt_upload_file('spec/support/common/hiera-eyaml', '/etc/eyaml')
  end

  desc 'Install Agent'
  task :install_agent do
    master_uri = master.uri
    agent_uri = agent.uri

    puts "GH: master=#{master_uri}"
    puts "GH: agent=#{agent_uri}"

    task_prefix = "bundle exec bolt task run --modulepath spec/fixtures/modules --target #{agent_uri}"

    system("#{task_prefix(agent_uri)} puppet_agent::install install_options='REINSTALLMODE=\"amus\" PUPPET_AGENT_STARTUP_MODE=Manual'")
    system("#{task_prefix(agent_uri)} puppet_conf action='set' setting='server' value='#{master_uri}'")

    # agent.run_shell('/opt/puppetlabs/bin/puppet agent -t')
  end

  desc 'Puppet agent run'
  task :puppet_agent_run do
    agent.run_shell('/opt/puppetlabs/bin/puppet agent -t', expect_failures: true )
  end

  desc 'Reload PE server'
  task :reload_pe do
    master.run_shell('/opt/puppetlabs/bin/puppetserver reload') 
  end
end

# Rake::Task['run_task'].invoke('puppet_conf', )
desc 'Run a puppet task'
task :run_task,[:taskname] do |_task, args|
  # command_to_run = "bolt task run puppet_agent::install --targets #{result['target']} --inventoryfile inventory.yaml --modulepath #{DEFAULT_CONFIG_DATA['modulepath']}"
  puts args[:taskname]
  puts args
end

task :my_task, [:arg1, :arg2] do |t, args|
  puts "Args were: #{args} of class #{args.class}"
  puts "arg1 was: '#{args[:arg1]}' of class #{args[:arg1].class}"
  puts "arg2 was: '#{args[:arg2]}' of class #{args[:arg2].class}"
end


namespace :acceptance do
  require 'puppet_litmus/rake_tasks'
  require_relative './spec/support/acceptance/helpers'
  include TargetHelpers

  desc 'Provisions the VMs. This is currently just the master'
  task :provision_vms do
    if File.exist?('inventory.yaml')
      # Check if a master VM's already been setup
      begin
        uri = master.uri
        puts("A master VM at '#{uri}' has already been set up")
        next
      rescue TargetNotFoundError
        # Pass-thru, this means that we haven't set up the master VM
      end
    end

    provision_list = ENV['PROVISION_LIST'] || 'acceptance'
    Rake::Task['litmus:provision_list'].invoke(provision_list)
  end

  # TODO: This should be refactored to use the https://github.com/puppetlabs/puppetlabs-peadm
  # module for PE setup
  desc 'Sets up PE on the master'
  task :setup_pe do
    master.bolt_run_script('spec/support/acceptance/install_pe.sh')
    # Setup hiera-eyaml config
    master.run_shell('rm -rf /etc/eyaml')
    master.bolt_upload_file('spec/support/common/hiera-eyaml', '/etc/eyaml')
  end

  desc 'Sets up the ServiceNow instance'
  task :setup_servicenow_instance, [:sn_instance, :sn_user, :sn_password, :sn_token] do |_, args|
    instance, user, password, token = servicenow_params(args)
    if instance.nil?
      # Start the mock ServiceNow instance. If an instance has already been started,
      # then the script will remove the old instance before replacing it with the new
      # one.
      puts("Starting the mock ServiceNow instance at the master (#{master.uri})")
      master.bolt_upload_file('./spec/support/acceptance/servicenow', '/tmp/servicenow')
      master.bolt_run_script('spec/support/acceptance/start_mock_servicenow_instance.sh')
      instance, user, password, token = "#{master.uri}:1080", 'mock_user', 'mock_password', 'mock_token'
    else
      # User provided their own ServiceNow instance so make sure that they've also
      # included the instance's credentials
      # Oauth tests will be skipped if a token is not provided.
      raise 'The ServiceNow username must be provided' if user.nil?
      raise 'The ServiceNow password must be provided' if password.nil?
      puts "oauth token not provided so the oauth token tests will be skipped" if token.nil?
    end

    # Update the inventory file
    puts('Updating the inventory.yaml file with the ServiceNow instance credentials')
    inventory_hash = LitmusHelpers.inventory_hash_from_inventory_file
    servicenow_group = inventory_hash['groups'].find { |g| g['name'] =~ %r{servicenow} }
    unless servicenow_group
      servicenow_group = { 'name' => 'servicenow_nodes' }
      inventory_hash['groups'].push(servicenow_group)
    end
    servicenow_group['targets'] = [{
      'uri' => instance,
      'config' => {
        'transport' => 'remote',
        'remote' => {
          'user' => user,
          'password' => password,
          'oauth_token' => token,
        }
      },
      'vars' => {
        'roles' => ['servicenow_instance'],
      }
    }]
    write_to_inventory_file(inventory_hash, 'inventory.yaml')
  end

  desc 'Installs the module on the master'
  task :install_module do
    Rake::Task['litmus:install_module'].invoke(master.uri)
  end

  desc 'Reloads puppetserver on the master'
  task :reload_module do
    result = master.run_shell('/opt/puppetlabs/bin/puppetserver reload').stdout.chomp
    puts "Error: #{result}" unless result.nil?
  end

  desc 'Gets the puppetserver logs for service now'
  task :get_logs do
    puts master.run_shell('tail -500 /var/log/puppetlabs/puppetserver/puppetserver.log').stdout.chomp
  end

  desc 'Do an agent run'
  task :agent_run do
    puts master.run_shell('puppet agent -t').stdout.chomp
  end

  desc 'Runs the tests'
  task :run_tests do
    rcommand  = 'bundle exec rspec ./spec/acceptance --format documentation'
    rspec_command += ' --format RspecJunitFormatter --out rspec_junit_results.xml' if ENV['CI'] == 'true'
    puts("Running the tests ...\n")
    unless system(rspec_command)
      # system returned false which means rspec failed. So exit 1 here
      exit 1
    end
  end

  desc 'Set up the test infrastructure'
  task :setup do
    tasks = [
      :provision_vms,
      :setup_pe,
      :setup_servicenow_instance,
      :install_module,
    ]

    tasks.each do |task|
      task = "acceptance:#{task}"
      puts("Invoking #{task}")
      Rake::Task[task].invoke
      puts("")
    end
  end

  desc 'Teardown the setup'
  task :tear_down do
    puts("Tearing down the test infrastructure ...\n")
    Rake::Task['litmus:tear_down'].invoke(master.uri)
    FileUtils.rm_f('inventory.yaml')
  end

  desc 'Task for CI'
  task :ci_run_tests do
    begin
      Rake::Task['acceptance:setup'].invoke
      Rake::Task['acceptance:run_tests'].invoke 
    ensure
      Rake::Task['acceptance:tear_down'].invoke
    end
  end
end

# Build the gem
desc 'Build the gem'
task :build do
  gemspec_path = File.join(Dir.pwd, 'common_events_library.gemspec')
  run_local_command("bundle exec gem build '#{gemspec_path}'")
end

# Tag the repo with a version in preparation for the release
#
# @param :version [String] a semantic version to tag the code with
# @param :sha [String] the sha at which to apply the version tag
desc 'Tag the repo with a version in preparation for release'
task :tag, [:version, :sha] do |_task, args|
  raise "Invalid version #{args[:version]} - must be like '1.2.3'" unless args[:version] =~ /^\d+\.\d+\.\d+$/

  run_local_command('git fetch upstream')
  run_local_command("git tag -a #{args[:version]} -m #{args[:version]} #{args[:sha]}")
  run_local_command('git push upstream --tags')
end

# Push the built gem to RubyGems
#
# @param :path [String] optional, the full or relative path to the built gem to be pushed
desc 'Push to RubyGems'
task :push, [:path] do |_task, args|
  raise 'No discoverable gem for pushing' if Dir.glob("common_events_library*\.gem").empty? && args[:path].nil?
  raise "No file found at specified path: '#{args[:path]}'" unless File.exist?(args[:path])

  path = args[:path] || File.join(Dir.pwd, Dir.glob("common_events_library*\.gem")[0])
  run_local_command("bundle exec gem push #{path}")
end

# Create a file at ~/.gem/credentials with artifactory credentials
#
# Requires environment variables LDAP_USERNAME, and LDAP_PASSWORD to create the credentials file.
desc 'Set up Artifactory Credentials'
task :artifactory_creds do
  run_local_command("mkdir -p ~/.gem")
  run_local_command("touch ~/.gem/credentials")
  run_local_command("chmod 0600 ~/.gem/credentials")
  run_local_command("curl -u#{ENV["LDAP_USERNAME"]}:#{ENV["LDAP_PASSWORD"]} https://artifactory.delivery.puppetlabs.net/artifactory/api/gems/rubygems/api/v1/api_key.yaml > ~/.gem/credentials")
end

# Push the built gem to Artifactory using credentials in ~/.gem/credentials
desc 'Push to Artifactory'
task :push_to_artifactory do
  raise 'No discoverable gem for pushing' if Dir.glob("common_events_library*\.gem").empty? && args[:path].nil?

  path = File.join(Dir.pwd, Dir.glob("common_events_library-#{CommonEventsLibrary::VERSION}.gem")[0])
  run_local_command("gem push #{path} --host https://artifactory.delivery.puppetlabs.net/artifactory/api/gems/rubygems")
end
