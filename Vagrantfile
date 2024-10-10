# vagrant up --provider=aws
# vagrant destroy -f && vagrant up --provider=aws

TOPIC_NAME = "errors"
ACCOUNT_ID = "992382682634"
AWS_REGION = "eu-west-1"
MAIN_SH_ARGS = <<MARKER
-e "playbook_name=ansible-loki discord_message_owner_name=#{Etc.getpwuid(Process.uid).name}"
MARKER
NODE_COUNT = 1
Vagrant.configure("2") do |config|
  (1..NODE_COUNT).each do |i|
    config.vm.define "node#{i}" do |subconfig|
      subconfig.vm.provision "shell", inline: <<-SHELL
        set -euxo pipefail
        echo "start vagrant file"
        cd /vagrant
        python3 -m venv /tmp/ansibleenv
        source /tmp/ansibleenv/bin/activate
        aws s3 cp s3://resource-pension-stg/get-pip.py - | python3
        echo $PWD
        export VAULT_PASSWORD=#{`op read "op://Security/ansible-vault tamal-pension-stg/password"`.strip!}
        echo "$VAULT_PASSWORD" > vault_password
        export ANSIBLE_VERBOSITY=0
        export ANSIBLE_DISPLAY_SKIPPED_HOSTS=false
        if [ -f "main.sh" ]; then
          echo "Local main.sh found. Run the local main.sh script..."
          bash main.sh #{MAIN_SH_ARGS}
        else
          echo "Local main.sh not found. running the main.sh script from the URL..."
          curl -s https://raw.githubusercontent.com/inqwise/ansible-automation-toolkit/default/main_amzn2023.sh | bash -s -- #{MAIN_SH_ARGS}
        fi
        rm vault_password
      SHELL

      subconfig.vm.provider :aws do |aws, override|
        override.vm.box = "dummy"
        override.ssh.username = "ec2-user"
        override.ssh.private_key_path = "~/.ssh/id_rsa"
        aws.aws_dir = ENV['HOME'] + "/.aws/"
        aws.aws_profile = "pension-stg"

        aws.keypair_name = Etc.getpwuid(Process.uid).name
        override.vm.allowed_synced_folder_types = [:rsync]
        override.vm.synced_folder ".", "/vagrant", type: :rsync, rsync__exclude: ['.git/','inqwise/'], disabled: false
        #common_collection_path = ENV['COMMON_COLLECTION_PATH'] || '~/git/ansible-common-collection' 
        #stacktrek_collection_path = ENV['STACKTREK_COLLECTION_PATH'] || '~/git/ansible-stack-trek'
        #override.vm.synced_folder common_collection_path, '/vagrant/collections/ansible_collections/inqwise/common', type: :rsync, rsync__exclude: '.git/', disabled: false
        #override.vm.synced_folder stacktrek_collection_path, '/vagrant/collections/ansible_collections/inqwise/stacktrek', type: :rsync, rsync__exclude: '.git/', disabled: false

        aws.region = AWS_REGION  
        aws.security_groups = ["sg-077f8d7d58d420467","sg-01095054cdf814468"]
        # public-ssh, loki
        aws.ami = "ami-0ad71aff41c394ba8"
        aws.instance_type = "t4g.small"
        aws.subnet_id = "subnet-0331d92e81f166c9f"
        aws.associate_public_ip = true
        aws.iam_instance_profile_name = "bootstrap-role"
        aws.tags = {
          Name: "loki-test-#{Etc.getpwuid(Process.uid).name}",
          private_dns: "loki-test-#{Etc.getpwuid(Process.uid).name}"
        }
      end
    end
  end
end 