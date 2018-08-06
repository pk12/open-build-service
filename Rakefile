CONTAINER_USERID = %x(id -u).freeze
VERSION = '42.3'.freeze

namespace :docker do
  desc 'Build our development environment'
  task :build do
    begin
      # Setup the git commit message template
      sh 'git config commit.template .gitmessage'
      sh 'touch docker-files/home/.bash_history docker-files/home/.irb_history'
      sh "cat << EOF > docker-compose.override.yml
# This file is generated by our Rakefile. Do not change it!
version: '2.1'
services:
  frontend:
    build:
      args:
        CONTAINER_USERID: #{CONTAINER_USERID}
    volumes:
      - .:/obs
      - ./docker-files/home/.bash_history:/home/frontend/.bash_history
      - ./docker-files/home/.irb_history:/home/frontend/.irb_history
      - ./docker-files/home/.irbrc:/home/frontend/.irbrc

EOF"
      # Build the frontend container and pull newer version of the image if available
      sh 'docker-compose build --pull frontend'
      # Build the minitest container on top of that
      sh 'docker-compose -f docker-compose.yml -f docker-compose.minitest.yml build minitest'
      # Bootstrap the app
      sh 'docker-compose up -d db'
      sh 'docker-compose run --no-deps --rm frontend bundle exec rake dev:bootstrap RAILS_ENV=development'
    ensure
      sh 'docker-compose stop'
    end
  end

  namespace :maintainer do
    def tags_for(container_type)
      "-t openbuildservice/#{container_type}:#{VERSION} -t openbuildservice/#{container_type}"
    end

    desc 'Rebuild all our static containers'
    multitask rebuild: ['rebuild:all'] do
    end
    namespace :rebuild do
      multitask all: [:base, :backend, 'frontend-base', :mariadb, :memcached] do
      end
      task :base do
        sh "docker build docker-files/base/ #{tags_for(:base)} -f docker-files/base/Dockerfile.#{VERSION}"
      end
      task mariadb: [:base] do
        sh "docker build docker-files/mariadb/ #{tags_for(:mariadb)} -f docker-files/mariadb/Dockerfile.mariadb"
      end
      task memcached: [:base] do
        sh "docker build docker-files/memcached/ #{tags_for(:memcached)} -f docker-files/memcached/Dockerfile.memcached"
      end
      task 'frontend-base' => [:base] do
        sh "docker build src/api/ #{tags_for('frontend-base')} -f src/api/docker-files/Dockerfile.frontend-base"
      end
      task 'frontend-backend' => [:base] do
        sh "docker build src/api/ #{tags_for('frontend-backend')} -f src/api/docker-files/Dockerfile.frontend-backend"
      end
      task backend: [:base] do
        sh "docker build src/backend/ #{tags_for(:backend)} -f src/backend/docker-files/Dockerfile.backend"
      end
    end

    desc 'Rebuild and publish all our static containers'
    task publish: [:rebuild, 'publish:all'] do
    end
    namespace :publish do
      multitask all: [:base, :mariadb, :memcached, :backend, 'frontend-base'] do
      end
      task :base do
        sh "docker push openbuildservice/base:#{VERSION}"
        sh 'docker push openbuildservice/base'
      end
      task :mariadb do
        sh "docker push openbuildservice/mariadb:#{VERSION}"
        sh 'docker push openbuildservice/mariadb'
      end
      task :memcached do
        sh "docker push openbuildservice/memcached:#{VERSION}"
        sh 'docker push openbuildservice/memcached'
      end
      task :backend do
        sh "docker push openbuildservice/backend:#{VERSION}"
        sh 'docker push openbuildservice/backend'
      end
      task 'frontend-base' do
        sh "docker push openbuildservice/frontend-base:#{VERSION}"
        sh 'docker push openbuildservice/frontend-base'
      end
    end
  end
  namespace :ahm do
    desc 'Prepare the application health monitoring containers'
    task :prepare do
      begin
        sh 'docker-compose -f docker-compose.ahm.yml -f docker-compose.yml up -d rabbit'
        sh 'wget http://localhost:15672/cli/rabbitmqadmin -O contrib/rabbitmqadmin'
        sh 'chmod +x contrib/rabbitmqadmin'
        sh './contrib/rabbitmqadmin declare exchange name=pubsub type=topic durable=true auto_delete=false internal=false'
        # configure the app
        sh 'docker-compose -f docker-compose.ahm.yml -f docker-compose.yml up -d db'
        sh 'docker-compose -f docker-compose.ahm.yml -f docker-compose.yml run --no-deps --rm frontend bundle exec rake dev:ahm:configure'
      ensure
        sh 'docker-compose -f docker-compose.ahm.yml -f docker-compose.yml stop'
      end
    end
  end
end

def environment_vars(with_coverage = true)
  environment = []
  environment.concat(['-e', 'DO_COVERAGE=1']) if with_coverage && ENV['CIRCLECI']
  environment.concat(['-e', 'EAGER_LOAD=1'])
  environment.concat(['-e', "TEST_SUITE='#{ENV['TEST_SUITE']}'"])
  environment
end
