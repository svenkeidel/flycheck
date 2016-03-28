# Copyright (c) 2012-2016 Sebastian Wiesner and Flycheck contributors

# This program is free software: you can redistribute it and/or modify it under
# the terms of the GNU General Public License as published by the Free Software
# Foundation, either version 3 of the License, or (at your option) any later
# version.

# This program is distributed in the hope that it will be useful, but WITHOUT
# ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS
# FOR A PARTICULAR PURPOSE.  See the GNU General Public License for more
# details.

# You should have received a copy of the GNU General Public License along with
# this program.  If not, see <http://www.gnu.org/licenses/>.

require_relative 'admin/lib/flycheck/util'
require_relative 'admin/lib/flycheck/lint'

require 'rake'
require 'rake/clean'

include Flycheck::Util

SOURCES = FileList['flycheck.el', 'flycheck-ert.el', 'flycheck-buttercup.el']
OBJECTS = SOURCES.ext('.elc')

ALL_SOURCES = FileList.new(SOURCES) do |f|
  f.include('admin/*.el')
  f.include('test/*.el')
  f.include('test/specs/**/*.el')
end

DOC_SOURCES = FileList['doc/flycheck.texi']
DOC_SOURCES.add('doc/*.texi')

MARKDOWN_SOURCES = FileList['*.md']

RUBY_SOURCES = FileList['Rakefile', 'admin/lib/**/*.rb']

# File tasks and rules
file 'doc/images/logo.png' => ['flycheck.svg'] do |t|
  sh 'convert', t.prerequisites.first,
     '-trim', '-background', 'white',
     '-bordercolor', 'white', '-border', '5',
     t.name
  sh 'optipng', t.name
end

rule '.elc', [:build_flags] => ['.el'] do |t, args|
  batch_args = ['-L', '.']
  if args.to_a.include? 'error-on-warn'
    batch_args << '--eval'
    batch_args << '(setq byte-compile-error-on-warn t)'
  end

  batch_args << '-f'
  batch_args << 'batch-byte-compile'
  batch_args += t.prerequisites

  sh 'cask', 'exec', *emacs_batch(*batch_args)
end

# Force rake to preserve the description of tasks
Rake::TaskManager.record_task_metadata = true

# Tasks
namespace :init do
  CLOBBER << '.cask/'

  desc 'Install all dependencies'
  task :deps do
    sh 'cask', 'install'
    sh 'cask', 'update'
  end

  desc 'Initialise the project'
  task all: [:deps]
end

namespace :verify do
  desc 'Verify Markdown documents'
  task :markdown do
    sh('mdl', '--style', 'admin/markdown_style', *MARKDOWN_SOURCES)
  end

  desc 'Verify Ruby sources'
  task :ruby do
    sh('rubocop', '--config', '.rubocop.yml', *RUBY_SOURCES)
  end

  desc 'Verify Emacs Lisp sources'
  task :elisp do
    Flycheck::Lint.check_files(ALL_SOURCES.to_a)
  end

  desc 'Verify all source files'
  task all: [:markdown, :ruby, :elisp]
end

namespace :generate do
  desc 'Generate the logo'
  task logo: 'doc/images/logo.png'
end

namespace :compile do
  CLEAN.add(OBJECTS)

  desc 'Compile byte code'
  task :elc, [:build_flags] => OBJECTS

  desc 'Compile everything'
  task :all, [:build_flags] => [:elc]
end

namespace :test do
  def spec_task(name, directory)
    task name, [:pattern] => OBJECTS do |_, args|
      command = ['cask', 'exec', 'buttercup', '-L', '.', directory]
      command += ['--pattern', args.pattern] if args.pattern
      sh(*command)
    end
  end

  namespace :unit do
    desc 'Run unit specs'
    spec_task :specs, 'test/specs'

    desc 'Run ERT unit tests'
    task :ert, [:selector] => OBJECTS do |_, args|
      selector = '(not (tag external-tool))'
      selector = "(and #{selector} #{args.selector})" if args.selector
      sh(*emacs_batch('--load', 'test/run.el', '-f', 'flycheck-run-tests-main',
                      selector))
    end

    desc 'Run all unit tests'
    task all: [:specs, :ert]
  end

  namespace :integration do
    desc 'Run integration specs'
    spec_task :specs, 'test/integration'

    desc 'Run integration tests'
    task :ert, [:selector] => OBJECTS do |_, args|
      selector = '(tag external-tool)'
      selector = "(and #{selector} #{args.selector})" if args.selector
      sh(*emacs_batch('--load', 'test/run.el',
                      '-f', 'flycheck-run-tests-main',
                      selector))
    end

    desc 'Run all integration tests'
    task all: [:specs, :ert, :doc]
  end

  desc 'Run all buttercup specs (unit and integration)'
  task :specs, [:pattern] => ['unit:specs', 'integration:specs']

  desc 'Run all tests'
  task all: ['unit:all', 'integration:all']
end

namespace :dist do
  CLEAN << 'dist/'

  desc 'Build an Emacs package'
  task :package do
    sh 'cask', 'package'
  end

  desc 'Build all distributions'
  task all: [:package]
end

# Top-level targets
namespace :check do
  desc 'Check Flycheck for a given LANGUAGE'
  task :language, [:language] => 'verify:elisp' do |_, args|
    task('test:integration:ert').invoke("(language #{args.language})")
    task('test:unit:specs').invoke('^Manual')
  end

  desc 'Check Flycheck fast (verify, compile, unit tests only)'
  task fast: ['verify:all', 'compile:all', 'test:unit:all']

  desc 'Check Flycheck (verify, compile, test)'
  task all: ['verify:all', 'compile:all', 'test:all']

  desc 'Check documentation (generate and test)'
  task doc: ['doc:info'] do
    task('test:unit:specs').invoke('^Manual')
  end
end

task :help do
  tasks = Rake.application.tasks.select(&:comment)
  width = tasks.map { |t| t.name_with_args.length }.max || 10

  task_list = tasks.map do |t|
    "- #{t.name_with_args.ljust(width)} - #{t.comment}"
  end.join("\n")

  puts <<EOF
# Flycheck rake

    rake <task>

Rake helps you automate various tasks in Flycheck.

# Required tools

Emacs and Cask (http://cask.readthedocs.org/).  Optionally, also:

- Rubocop (https://github.com/bbatsov/rubocop)
- Markdownlint (https://github.com/mivok/markdownlint)

# Task list

#{task_list}
EOF
end

task default: :help
