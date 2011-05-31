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

      desc "cook #{cookbook}"
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
          metadata = JSON.parse(git_output("show #{rev}:metadata.json")) rescue {}
          if metadata['version'] && metadata['version'] != version
            version = metadata['version']
            git "tag -a #{version}  -m 'Chef cookbook #{cookbook} version: #{version}' #{rev}"
          end
        end

        create_repo(cookbook) unless repo_exists?(cookbook)

        # push!
        git "remote rm origin"
        git "remote add cookbooks git@github.com:cookbooks/#{cookbook}.git"
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

def create_repo(name)
  sleep 1 # github api throttlin'
  post = {
    :login => config['login'],
    :token => config['token'],
    :public => 1,
    :name => "cookbooks/#{name}",
    :description => "A Chef cookbook for #{name}",
    :homepage => "https://github.com/opscode/cookbooks/blob/master/LICENSE"
  }
  RestClient.post "https://github.com/api/v2/json/repos/create", post.to_json, :content_type => :json, :accept => :json
end

def repo_exists?(name)
  sleep 1 # github api throttlin'
  basic_auth = Base64.encode64("#{config['login']}/token:#{config['token']}").gsub("\n", '')
  headers = { 'Authorization' => "Basic #{basic_auth}"}
  repositories = JSON.parse(RestClient.get("https://github.com/api/v2/json/organizations/repositories", headers))
  repositories['repositories'].detect { |r| r["name"] == name }
end

task :submodule do
  git "submodule update --init"
  Dir.chdir("#{Rake.original_dir}/cookbooks")
  git "pull origin master"
  Dir.chdir(Rake.original_dir)
  git "commit cookbooks -m '#{`whoami`.chomp} updated cookbooks at #{`date`.chomp}'"
end

task :default => :submodule do
  cookbook_list.each { |cookbook| Rake::Task["tmp/#{cookbook}"].invoke }
end

cookbook_list