require File.expand_path(File.join(File.dirname(__FILE__), "spec/jasmine_helper.rb"))

def jasmine_sources
  sources  = ["src/base.js", "src/util.js", "src/Env.js", "src/Reporter.js", "src/Block.js"]
  sources += Dir.glob('src/*.js').reject { |f| f == 'src/base.js' || sources.include?(f) }.sort
  sources
end

def jasmine_html_sources
  ["src/html/TrivialReporter.js"]
end

def jasmine_filename
  "jasmine-#{jasmine_version}.js"
end

def jasmine_version
  "#{version_hash['major']}.#{version_hash['minor']}.#{version_hash['build']}"
end

def version_hash
  require 'json'
  @version ||= JSON.parse(File.new("src/version.json").read);
end

def start_jasmine_server(jasmine_includes = nil)
  require File.expand_path(File.join(JasmineHelper.jasmine_root, "contrib/ruby/jasmine_spec_builder"))

  puts "your tests are here:"
  puts "  http://localhost:8888/run.html"

  Jasmine::SimpleServer.start(
      8888,
      lambda { JasmineHelper.specs },
      JasmineHelper.dir_mappings,
      :jasmine_files => jasmine_includes)
end

task :default => 'jasmine:dist'

def substitute_jasmine_version(filename)
  contents = File.read(filename)
  contents = contents.gsub(/##JASMINE_VERSION##/, (jasmine_version))
  File.open(filename, 'w') { |f| f.write(contents) }
end

namespace :jasmine do

  desc 'Prepares for distribution'
  task :dist => ['jasmine:build', 'jasmine:doc', 'jasmine:build_example_project']

  desc 'Check jasmine sources for coding problems'
  task :lint do
    passed = true
    jasmine_sources.each do |src|
      lines = File.read(src).split(/\n/)
      lines.each_index do |i|
        line = lines[i]
        undefineds = line.scan(/.?undefined/)
        if undefineds.include?(" undefined") || undefineds.include?("\tundefined")
          puts "Dangerous undefined at #{src}:#{i}:\n > #{line}"
          passed = false
        end

        if line.scan(/window/).length > 0
          puts "Dangerous window at #{src}:#{i}:\n > #{line}"
          passed = false
        end
      end
    end

    unless passed
      puts "Lint failed!"
      exit 1
    end
  end

  desc 'Builds lib/jasmine from source'
  task :build => :lint do
    puts 'Building Jasmine from source'

    sources = jasmine_sources
    version = version_hash

    old_jasmine_files = Dir.glob('lib/jasmine*.js')
    old_jasmine_files.each { |file| File.delete(file) }

    File.open("lib/jasmine.js", 'w') do |jasmine|
      sources.each do |source_filename|
        jasmine.puts(File.read(source_filename))
      end

      jasmine.puts %{
jasmine.version_= {
  "major": #{version['major']},
  "minor": #{version['minor']},
  "build": #{version['build']},
  "revision": #{Time.now.to_i}
};
}
    end

    File.open("lib/jasmine-html.js", 'w') do |jasmine_html|
      jasmine_html_sources.each do |source_filename|
        jasmine_html.puts(File.read(source_filename))
      end
    end

    FileUtils.cp("src/html/jasmine.css", "lib/jasmine.css")
  end

  desc "Build jasmine documentation"
  task :doc do
    puts 'Creating Jasmine Documentation'
    require 'rubygems'
    #sudo gem install ragaskar-jsdoc_helper
    require 'jsdoc_helper'


    JsdocHelper::Rake::Task.new(:lambda_jsdoc) do |t|
      t[:files] = jasmine_sources << jasmine_html_sources
      t[:options] = "-a"
    end
    Rake::Task[:lambda_jsdoc].invoke
  end

  desc "Build example project"
  task :build_example_project do
    require 'tmpdir'

    temp_dir = File.join(Dir.tmpdir, 'jasmine-standalone-project')
    puts "Building Example Project in #{temp_dir}"
    FileUtils.rm_r temp_dir if File.exists?(temp_dir)
    Dir.mkdir(temp_dir)

    root = JasmineHelper.jasmine_root
    FileUtils.cp_r File.join(root, 'example/.'), File.join(temp_dir)
    substitute_jasmine_version(File.join(temp_dir, "SpecRunner.html"))

    lib_dir = File.join(temp_dir, "lib/jasmine-#{jasmine_version}")
    FileUtils.mkdir_p(lib_dir)
    {
        "lib/jasmine.js" => "jasmine.js",
        "lib/jasmine-html.js" => "jasmine-html.js",
        "src/html/jasmine.css" => "jasmine.css"
    }.each_pair do |src, dest|
      FileUtils.cp(File.join(root, src), File.join(lib_dir, dest))
    end

    dist = File.join(root, 'dist')
    FileUtils.rm_r dist if File.exists?(dist)
    Dir.mkdir(dist)
    exec "cd #{temp_dir} && zip -r #{File.join(dist, "jasmine-standalone-#{jasmine_version}.zip")} ."
  end


  task :server do
    files = jasmine_sources + jasmine_html_sources
    jasmine_includes = lambda {
      raw_jasmine_includes = files.collect { |f| File.expand_path(File.join(JasmineHelper.jasmine_root, f)) }
      Jasmine.cachebust(raw_jasmine_includes).collect { |f| f.sub(JasmineHelper.jasmine_src_dir, "/src").sub(JasmineHelper.jasmine_lib_dir, "/lib") }
    }
    start_jasmine_server(jasmine_includes)
  end

  task :server_build => 'jasmine:build' do

    start_jasmine_server
  end

  namespace :test do
    task :ci => :'ci:local'
    namespace :ci do

      task :local => 'jasmine:build' do
        require "spec"
        require 'spec/rake/spectask'
        Spec::Rake::SpecTask.new(:lambda_ci) do |t|
          t.spec_opts = ["--color", "--format", "specdoc"]
          t.spec_files = ["spec/jasmine_spec.rb"]
        end
        Rake::Task[:lambda_ci].invoke
      end

      task :saucelabs => ['jasmine:copy_saucelabs_config', 'jasmine:build'] do
        ENV['SAUCELABS'] = 'true'
        Rake::Task['jasmine:test:ci:local'].invoke
      end
    end
  end

  desc 'Copy saucelabs.yml to work directory'
  task 'copy_saucelabs_config' do
    FileUtils.cp '../saucelabs.yml', 'spec'
  end
end

task :jasmine => ['jasmine:server']
