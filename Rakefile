require 'rubygems'
require 'rake'

volatile_requires = ['rcov/rcovtask']
not_loaded = []
volatile_requires.each do |file|
  begin
    require file
  rescue LoadError
    not_loaded.push file
  end
end

# ----- Benchmarking -----

temp_desc = <<END
Benchmark haml against ERb.
  TIMES=n sets the number of runs. Defaults to 100.
END

desc temp_desc.chomp
task :benchmark do
  require 'test/benchmark'

  puts "Running benchmarks #{ENV['TIMES']} times..." if ENV['TIMES']
  times = ENV['TIMES'].to_i if ENV['TIMES']
  Haml.benchmark(times || 100)
  puts '-'*51
end

# Benchmarking gets screwed up if some other tasks have been
# initialized.
unless ARGV[0] == 'benchmark'

  # ----- Default: Testing ------

  desc 'Default: run unit tests.'
  task :default => :test

  require 'rake/testtask'

  Rake::TestTask.new do |t|
    t.libs << 'lib'
    t.pattern = 'test/**/*_test.rb'
    t.verbose = true
  end
  Rake::Task[:test].send(:add_comment, <<END)
To run with an alternate version of Rails, make test/rails a symlink to that version.
END

  # ----- Packaging -----

  require 'rake/gempackagetask'
  require 'lib/haml'

  # Before we run the package task,
  # we want to create a REVISION file
  # if we've checked out Haml from git
  # and we aren't building for a release.
  create_revision = Haml.version[:rev] && !Rake.application.top_level_tasks.include?('release')

  spec = Gem::Specification.new do |spec|
    spec.name = 'haml'
    spec.summary = "An elegant, structured XHTML/XML templating engine.\nComes with Sass, a similar CSS templating engine."
    spec.version = File.read('VERSION').strip
    spec.author = 'Hampton Catlin'
    spec.email = 'haml@googlegroups.com'
    spec.description = <<-END
      Haml (HTML Abstraction Markup Language) is a layer on top of XHTML or XML
      that's designed to express the structure of XHTML or XML documents
      in a non-repetitive, elegant, easy way,
      using indentation rather than closing tags
      and allowing Ruby to be embedded with ease.
      It was originally envisioned as a plugin for Ruby on Rails,
      but it can function as a stand-alone templating engine.
    END
    #'

    readmes = FileList.new('*') do |list|
      list.exclude(/(^|[^.a-z])[a-z]+/)
      list.exclude('TODO')
    end.to_a
    spec.executables = ['haml', 'html2haml', 'sass', 'css2sass']
    readmes << 'REVISION' if create_revision
    spec.files = FileList['lib/**/*', 'bin/*', 'test/**/*', 'Rakefile', 'init.rb'].to_a + readmes
    spec.autorequire = ['haml', 'sass']
    spec.homepage = 'http://haml.hamptoncatlin.com/'
    spec.has_rdoc = true
    spec.extra_rdoc_files = readmes
    spec.rdoc_options += [
      '--title', 'Haml',
      '--main', 'README.rdoc',
      '--exclude', 'lib/haml/buffer.rb',
      '--line-numbers',
      '--inline-source'
    ]
    spec.test_files = FileList['test/**/*_test.rb'].to_a
  end

  Rake::GemPackageTask.new(spec) do |pkg|
    pkg.need_zip     = true
    pkg.need_tar_gz  = true
    pkg.need_tar_bz2 = true
  end

  desc "This is an internal task."
  task :revision_file do
    File.open('REVISION', 'w') { |f| f.puts Haml.version[:rev] } if create_revision
  end
  Rake::Task[:package].prerequisites.insert(0, :revision_file)

  # We also need to get rid of this file after packaging.
  Rake::Task[:package].enhance { File.delete('REVISION') if File.exists?('REVISION') }

  task :install => [:package] do
    sudo = RUBY_PLATFORM =~ /win32/ ? '' : 'sudo'
    sh %{#{sudo} gem install --no-ri pkg/haml-#{File.read('VERSION').strip}}
  end

  task :release => [:package] do
    name, version = ENV['NAME'], ENV['VERSION']
    raise "Must supply NAME and VERSION for release task." unless name && version
    sh %{rubyforge login}
    sh %{rubyforge add_release haml haml "#{name} (v#{version})" pkg/haml-#{version}.gem}
    sh %{rubyforge add_file    haml haml "#{name} (v#{version})" pkg/haml-#{version}.tar.gz}
    sh %{rubyforge add_file    haml haml "#{name} (v#{version})" pkg/haml-#{version}.tar.bz2}
    sh %{rubyforge add_file    haml haml "#{name} (v#{version})" pkg/haml-#{version}.zip}
  end

  # ----- Documentation -----

  require 'rake/rdoctask'

  rdoc_task = Proc.new do |rdoc|
    rdoc.title    = 'Haml/Sass'
    rdoc.options << '--line-numbers' << '--inline-source'
    rdoc.rdoc_files.include('README.rdoc')
    rdoc.rdoc_files.include('lib/**/*.rb')
    rdoc.rdoc_files.exclude('lib/haml/buffer.rb')
    rdoc.rdoc_files.exclude('lib/haml/util.rb')
    rdoc.rdoc_files.exclude('lib/sass/tree/*')
  end

  Rake::RDocTask.new do |rdoc|
    rdoc_task.call(rdoc)
    rdoc.rdoc_dir = 'rdoc'
  end

  Rake::RDocTask.new(:rdoc_devel) do |rdoc|
    rdoc_task.call(rdoc)
    rdoc.rdoc_dir = 'rdoc_devel'
    rdoc.options << '--all'
    rdoc.rdoc_files.include('test/*.rb')

    # Get rid of exclusion rules
    rdoc.rdoc_files = Rake::FileList.new(*rdoc.rdoc_files.to_a)
    rdoc.rdoc_files.include('lib/haml/buffer.rb')
    rdoc.rdoc_files.include('lib/sass/tree/*')
  end

  # ----- Coverage -----

  unless not_loaded.include? 'rcov/rcovtask'
    Rcov::RcovTask.new do |t|
      t.libs << "test"
      t.test_files = FileList['test/**/*_test.rb']
      t.rcov_opts << '-x' << '"^\/"'
      if ENV['NON_NATIVE']
        t.rcov_opts << "--no-rcovrt"
      end
      t.verbose = true
    end
  end

  # ----- Profiling -----

  temp_desc = <<-END
  Run a profile of haml.
    ENGINE=str sets the engine to be profiled (Haml or Sass).
    TIMES=n sets the number of runs. Defaults to 100.
    FILE=n sets the file to profile. Defaults to 'standard'.
  END
  desc temp_desc.chomp
  task :profile do
    require 'test/profile'

    engine = ENV['ENGINE'] && ENV['ENGINE'].downcase == 'sass' ? Sass : Haml

    puts '-'*51, "Profiling #{engine}", '-'*51

    args = []
    args.push ENV['TIMES'].to_i if ENV['TIMES']
    args.push ENV['FILE'] if ENV['FILE']

    profiler = engine::Profiler.new
    res = profiler.profile(*args)
    puts res

    puts '-'*51
  end

end
