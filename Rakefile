require 'rubygems'
require 'bundler'
Bundler.setup

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

        # TODO: verify github repo is present, create if not

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