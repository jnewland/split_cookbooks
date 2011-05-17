task :submodule do
  git "submodule update --init"
end

require 'rake/clean'
CLEAN.include('tmp/*')

def git(command)
  system %{git #{command}}
end

cookbook_list = FileList['cookbooks/*'].map do |cookbook_path|

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
      git "remote rm origin"
      Dir.chdir(Rake.original_dir)
    end

    cookbook
  end
end.reject { |c| c.nil? }

task :default do
  cookbook_list.each { |cookbook| Rake::Task["tmp/#{cookbook}"].invoke }
end