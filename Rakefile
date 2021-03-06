desc "Runs the specs [EMPTY]"
task :spec do
  # Provide your own implementation
end

task :version do
  git_remotes = `git remote`.strip.split("\n")

  if git_remotes.count > 0
    puts "-- fetching version number from github"
    sh 'git fetch'

    remote_version = remote_spec_version
  end

  if remote_version.nil?
    puts "There is no current released version. You're about to release a new Pod."
    version = "0.0.1"
  else
    puts "The current released version of your pod is " + remote_spec_version.to_s()
    version = suggested_version_number
  end
  
  puts "Enter the version you want to release (" + version + ") "
  new_version_number = $stdin.gets.strip
  if new_version_number == ""
    new_version_number = version
  end

  replace_version_number(new_version_number)
end

desc "Release a new version of the Pod"
task :release do

  puts "* Running version"
  sh "rake version"
  
  curr_branch, rem_branch, remote = git_information
  
  unless ENV['SKIP_CHECKS']
    if curr_branch.length == 0
      $stderr.puts "[!] You need to be on a branch in order to be able to do a release."
      exit 1
    end

    if `git tag`.strip.split("\n").include?(spec_version)
      $stderr.puts "[!] A tag for version `#{spec_version}' already exists. Change the version in the podspec"
      exit 1
    end

    puts "You are about to release `#{spec_version}`, is that correct? [y/n]"
    exit if $stdin.gets.strip.downcase != 'y'
  end

  puts "* Running specs"
  sh "rake spec"
 
  puts "* Linting the podspec"
  sh "pod lib lint"

  # Then release
  
  # If we have no origin set (perhaps new branch) exit because we cannot push
  if remote.length == 0
    $stderr.puts "[!] You need to have a configured remote for the current branch."
    exit 1
  end
  
  sh "git commit #{podspec_path} CHANGELOG.md -m 'Release #{spec_version}'"
  sh "git tag -a #{spec_version} -m 'Release #{spec_version}'"
  sh "git push #{remote} #{rem_branch}"
  sh "git push #{remote} --tags"
  sh "pod push #{curr_branch} #{podspec_path}"
end

# @return The current branch
#
def current_branch
  %x[git symbolic-ref -q HEAD | sed -e 's|^refs/heads/||'].strip
end

# @return The remote branch configure for the given branch name
#
def remote_branch_for_branch(branch)
  remote = git_remote_for_branch(branch)
  %x[git branch -r | grep '#{remote}/#{branch}'].strip
end

# @return The configure remote for the given branch name
#
def git_remote_for_branch(branch)
  %x[git config --get 'branch.#{branch}.remote'].strip
end

# @return All git information needed to push/fetch 
#
def git_information
  branch = current_branch
  return branch, remote_branch_for_branch(branch), git_remote_for_branch(branch)
end

# @return [Pod::Version] The version as reported by the Podspec.
#
def spec_version
  require 'cocoapods'
  spec = Pod::Specification.from_file(podspec_path)
  spec.version
end

# @return [Pod::Version] The version as reported by the Podspec from remote.
#
def remote_spec_version
  require 'cocoapods-core'

  if spec_file_exist_on_remote?
    curr_branch, rem_branch, remote = git_information
    
    remote_spec = eval(`git show #{remote}/#{remote_branch}:#{podspec_path}`)
    remote_spec.version
  else
    nil
  end
end

# @return [Bool] If the remote repository has a copy of the podpesc file or not.
#
def spec_file_exist_on_remote?
  curr_branch, rem_branch, remote = git_information
  
  test_condition = `if git rev-parse --verify --quiet #{remote}/#{rem_branch}:#{podspec_path} >/dev/null;
  then
  echo 'true'
  else
  echo 'false'
  fi`

  'true' == test_condition.strip
end

# @return [String] The relative path of the Podspec.
#
def podspec_path
  podspecs = Dir.glob('*.podspec')
  if podspecs.count == 1
    podspecs.first
  else
    raise "Could not select a podspec"
  end
end

# @return [String] The suggested version number based on the local and remote version numbers.
#
def suggested_version_number
  if spec_version != remote_spec_version
    spec_version.to_s()
  else
    next_version(spec_version).to_s()
  end
end

# @param  [Pod::Version] version
#         the version for which you need the next version
#
# @note   It is computed by bumping the last component of the versino string by 1.
#
# @return [Pod::Version] The version that comes next after the version supplied.
#
def next_version(version)
  version_components = version.to_s().split(".");
  last = (version_components.last.to_i() + 1).to_s
  version_components[-1] = last
  Pod::Version.new(version_components.join("."))
end

# @param  [String] new_version_number
#         the new version number
#
# @note   This methods replaces the version number in the podspec file with a new version number.
#
# @return void
#
def replace_version_number(new_version_number)
  text = File.read(podspec_path)
  text.gsub!(/(s.version( )*= ")#{spec_version}(")/, "\\1#{new_version_number}\\3")
  File.open(podspec_path, "w") { |file| file.puts text }
end