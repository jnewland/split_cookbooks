require 'rubygems'
require 'bundler'
Bundler.require
require 'yaml'
require 'base64'
require 'rake/clean'
CLEAN.include('tmp/*')

# Dynamically build tasks for all directories in cookbooks/*
def cookbook_list
  @cookbook_list ||= FileList['cookbooks/*'].map do |cookbook_path|
    if File.directory?(cookbook_path)
      cookbook = cookbook_path.split("/").last

      file "tmp/#{cookbook}" do
        git "clone --no-hardlinks cookbooks tmp/#{cookbook}"
        Dir.chdir("#{Rake.original_dir}/tmp/#{cookbook}")
        git "filter-branch --subdirectory-filter #{cookbook} HEAD -- --all"
        git "reset --hard"
        git "gc --aggressive"
        git "prune"

        # tag versions
        revisions = git_output "rev-list --topo-order --branches"
        version = nil
        revisions.split(/\n/).each do |rev|
          metadata = parse_metadata(cookbook, rev)
          if metadata['version'] && metadata['version'] != version
            version = metadata['version']
            git "tag -a #{version}  -m 'Chef cookbook #{cookbook} version: #{version}' #{rev}"
          end
        end

        create_repo(cookbook) unless repo_exists?(cookbook)

        # push!
        git "remote rm origin"
        git "remote add cookbooks git@github.com:#{config['org']}/#{cookbook}.git"
        git "push --force --tags cookbooks master"
        Dir.chdir(Rake.original_dir)

        Rake::Task['clean'].invoke
      end

      cookbook
    end
  end.reject { |c| c.nil? }
end

def git(command)
  system %{git #{command}}
end

def git_output(command)
  `git #{command}`.chomp
end

def config
  @config ||= YAML.load_file(File.join(Rake.original_dir, 'config.yml'))
end

def post(uri, payload)
  sleep 1 # github api throttlin'
  basic_auth = Base64.encode64("#{config['login']}/token:#{config['token']}").gsub("\n", '')
  headers = { 'Authorization' => "Basic #{basic_auth}", :content_type => :json, :accept => :json}
  JSON.parse(RestClient.post(uri, payload.to_json, headers))
end

def get(uri)
  sleep 1 # github api throttlin'
  basic_auth = Base64.encode64("#{config['login']}/token:#{config['token']}").gsub("\n", '')
  headers = { 'Authorization' => "Basic #{basic_auth}"}
  JSON.parse(RestClient.get(uri, headers))
end

def create_repo(name)
  repo_info = {
    :public      => 1,
    :name        => "#{config['org']}/#{name}",
    :description => "A Chef cookbook for #{name}",
    :homepage    => "https://github.com/opscode/cookbooks/blob/master/LICENSE"
  }
  post "https://github.com/api/v2/json/repos/create", repo_info
  post "https://github.com/api/v2/json/teams/#{config['team_id']}/repositories", {:name => "#{config['org']}/#{name}"}
end

def repo_exists?(name)
  repositories = get("https://github.com/api/v2/json/organizations/repositories")
  repositories['repositories'].detect { |r| r["name"] == name }
end

def parse_metadata(cookbook, rev)
  begin
    metadata = JSON.parse(git_output("show #{rev}:./metadata.json"))
  rescue
    puts "Generating metadata.json file.\nGit revision #{rev}"
    `knife cookbook metadata from file metadata.rb`
    metadata= JSON.parse(::File.read('metadata.json'))
    File.rm('metadata.json')
    puts "Cookbook #{cookbook} Version: #{metadata['version']}"
  end
  metadata
end

task :submodule do
  git "submodule update --init"
  Dir.chdir("#{Rake.original_dir}/cookbooks")
  git "pull origin master"
  Dir.chdir(Rake.original_dir)
  git "commit cookbooks -m 'updated upstream cookbooks at #{`date`.chomp}'"
end

desc "Update all cookbooks in Opscode's Chef Cookbooks repository."
task :default => [:submodule, :create_tasks] do
  begin
    cookbook_list.each { |cookbook| Rake::Task["tmp/#{cookbook}"].invoke }
  ensure
    Rake::Task['clean'].invoke
  end
end

task :create_tasks do
  cookbook_list
end

desc "Update specific cookbook (default: all) from Opscode's Chef Cookbooks repository."
task :update, [:cookbook] => [:submodule, :create_tasks] do |tsk, args|
  begin
    args.with_defaults(:cookbook => 'all')
    if args[:cookbook] == 'all'
      cookbook_list.each { |cookbook| Rake::Task["tmp/#{cookbook}"].invoke }
    else
      Rake::Task["tmp/#{args[:cookbook]}"].invoke
    end
  ensure
    Rake::Task['clean'].invoke
  end
end

cookbook_list