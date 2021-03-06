# Makefile

CONFIGURATION = master
CONSOLE = manager
ENVIRONMENT ?= betacloud
USERNAME = dragon

OPENSTACK = openstack
TERRAFORM = terraform

RESOURCE = openstack_networking_floatingip_v2.manager_floating_ip
STATE = errored.tfstate

ifneq (,$(wildcard ./local.env))
  include local.env
endif

export OS_CLOUD ?= $(ENVIRONMENT)

export TF_VAR_cloud_provider=$(ENVIRONMENT)

ifneq (,$(wildcard ./minio.env))
  include minio.env
endif

init:
	@if [ ! -d .terraform/providers ]; then $(TERRAFORM) init; fi

	@$(TERRAFORM) workspace select ${ENVIRONMENT} || $(TERRAFORM) workspace new ${ENVIRONMENT}

	@rm -f *_override.tf
	@for over in `grep '^# *override:' "environments/${ENVIRONMENT}.tfvars" | sed 's/^# *override://'`; do \
		echo $$over; \
		cp -p overrides/$${over}_override.tf .; \
	done

attach: init
	@$(TERRAFORM) import -var-file="environments/$(ENVIRONMENT).tfvars" $(RESOURCE) $(PARAMS)

detach: init
	@$(TERRAFORM) state rm $(RESOURCE) $(PARAMS)

state-push: init
	@$(TERRAFORM) state push $(STATE)

dry-run: init
	@$(TERRAFORM) plan -var-file="environments/$(ENVIRONMENT).tfvars" $(PARAMS)

deploy: init
	@touch .deploy.$(ENVIRONMENT)
	@$(TERRAFORM) apply -auto-approve -var-file="environments/$(ENVIRONMENT).tfvars" -var "configuration_version=$(CONFIGURATION)" $(PARAMS)

deploy-identity: init
	@touch .deploy.$(ENVIRONMENT)
	@$(TERRAFORM) apply -auto-approve -var-file="environments/$(ENVIRONMENT).tfvars" -var 'deploy_identity=true' $(PARAMS)

deploy-infra: init
	@touch .deploy.$(ENVIRONMENT)
	@$(TERRAFORM) apply -auto-approve -var-file="environments/$(ENVIRONMENT).tfvars" -var 'deploy_infrastructure=true' $(PARAMS)

deploy-ceph: init
	@touch .deploy.$(ENVIRONMENT)
	@$(TERRAFORM) apply -auto-approve -var-file="environments/$(ENVIRONMENT).tfvars" -var 'deploy_ceph=true' $(PARAMS)

deploy-openstack: init
	@touch .deploy.$(ENVIRONMENT)
	@$(TERRAFORM) apply -auto-approve -var-file="environments/$(ENVIRONMENT).tfvars" -var 'deploy_infrastructure=true' -var 'deploy_ceph=true' -var 'deploy_openstack=true' $(PARAMS)

deploy-full: init
	@touch .deploy.$(ENVIRONMENT)
	@$(TERRAFORM) apply -auto-approve -var-file="environments/$(ENVIRONMENT).tfvars" -var 'deploy_infrastructure=true' -var 'deploy_ceph=true' -var 'deploy_openstack=true' -var 'deploy_monitoring=true' $(PARAMS)

show: init
	@$(TERRAFORM) show

clean: init
	@$(TERRAFORM) destroy -auto-approve -var-file="environments/$(ENVIRONMENT).tfvars" $(PARAMS)
	@rm -f .deploy.$(ENVIRONMENT) .MANAGER_ADDRESS.$(ENVIRONMENT)
	@rm -f .id_rsa.$(ENVIRONMENT)
	@$(TERRAFORM) workspace select default
	@$(TERRAFORM) workspace delete $(ENVIRONMENT)
	@rm -f *_override.tf

list: init
	@$(TERRAFORM) state list

ssh: .MANAGER_ADDRESS.$(ENVIRONMENT) .id_rsa.$(ENVIRONMENT)
	@. ./.MANAGER_ADDRESS.$(ENVIRONMENT); \
	ssh -o StrictHostKeyChecking=no -i .id_rsa.$(ENVIRONMENT) $(USERNAME)@$$MANAGER_ADDRESS

sshuttle: .MANAGER_ADDRESS.$(ENVIRONMENT) .id_rsa.$(ENVIRONMENT)
	@. ./.MANAGER_ADDRESS.$(ENVIRONMENT); \
	sshuttle --ssh-cmd "ssh -o StrictHostKeyChecking=no -i .id_rsa.$(ENVIRONMENT) " -r $(USERNAME)@$$MANAGER_ADDRESS 192.168.16.0/20 192.168.32.0/20 192.168.96.0/20 192.168.112.0/20

console: .deploy.$(ENVIRONMENT)
	@$(OPENSTACK) console url show testbed-$(CONSOLE)

log: .deploy.$(ENVIRONMENT)
	@$(OPENSTACK) console log show testbed-$(CONSOLE)

.deploy.$(ENVIRONMENT): init
	@STAT=$$($(TERRAFORM) state list); \
	if test -n "$$STAT"; then touch .deploy.$(ENVIRONMENT); else echo "use make deploy or deploy-infra or ...."; exit 1; fi

.MANAGER_ADDRESS.$(ENVIRONMENT): .deploy.$(ENVIRONMENT)
	@MANAGER_ADDRESS=$$($(TERRAFORM) output manager_address); \
	echo "MANAGER_ADDRESS=$$MANAGER_ADDRESS" > $@;

.id_rsa.$(ENVIRONMENT): .deploy.$(ENVIRONMENT)
	@PRIVATE_KEY=$$($(TERRAFORM) output private_key); \
	echo "$$PRIVATE_KEY" > $@; \
        chmod 0600 $@

watch: .id_rsa.$(ENVIRONMENT) .MANAGER_ADDRESS.$(ENVIRONMENT)
	@. ./.MANAGER_ADDRESS.$(ENVIRONMENT); \
	DISP=0; \
	if test "$$COLORTERM" = "1"; then \
	  GREEN=$$(echo -e "\e[0;32m"); GREENBOLD=$$(echo -e "\e[1;32m"); BOLD=$$(echo -e "\e[0;1m"); RED=$$(echo -e "\e[0;31m"); YELLOW=$$(echo -e "\e[0;33m"); NORM=$$(echo -e "\e[0;0m"); \
	fi; \
	while true; do \
		LEN=$$(ssh -o StrictHostKeyChecking=no -i .id_rsa.$(ENVIRONMENT) ubuntu@$$MANAGER_ADDRESS sudo wc -l /var/log/cloud-init-output.log 2>/dev/null); \
		LEN=$${LEN%% *}; \
		if test -n "$$LEN" -a "$$LEN" != "$$DISP"; then \
		  OUT=$$(ssh -o StrictHostKeyChecking=no -i .id_rsa.$(ENVIRONMENT) ubuntu@$$MANAGER_ADDRESS sudo tail -n $$((LEN-DISP)) /var/log/cloud-init-output.log 2>/dev/null); \
		  echo -e "$$OUT" | sed -e "s/^\(TASK.*\)$$/$$BOLD\1$$NORM/" -e "s/^\(PLAY.*\)$$/$$GREEN\1$$NORM/" -e "s/^\(The system is finally up.*\)$$/$$GREENBOLD\1$$NORM/" -e "s/\(FAILED\)/$$RED\1$$NORM/g" -e "s/\(failed=[1-9][0-9]*\|unreachable=[1-9][0-9]*\)/$$RED\1$$NORM/g" -e "s/\(warn\|WARN\|RETRYING\)/$$YELLOW\1$$NORM/" -e "s/\(ok:\|ok=[0-9]*\)/$$GREEN\1$$NORM/"; \
		  if echo "$$OUT" | grep '^The system is finally up' >/dev/null 2>&1; then break; fi; \
		  DISP=$$LEN; \
		  sleep 5; \
		fi; \
	done;
	#@$(OPENSTACK) server list

ping: .MANAGER_ADDRESS.$(ENVIRONMENT)
	@. ./.MANAGER_ADDRESS.$(ENVIRONMENT); \
	ping -c3 $$MANAGER_ADDRESS

openstack: init
	@$(OPENSTACK)

endpoints: init
	@. ./.MANAGER_ADDRESS.$(ENVIRONMENT); \
	echo "| ARA        | http://$$MANAGER_ADDRESS:8120  |"; \
	echo "| Cockpit    | https://$$MANAGER_ADDRESS:8130 |"; \
	echo "| phpMyAdmin | http://$$MANAGER_ADDRESS:8110  |";

create: deploy
login: ssh
tunnel: sshuttle

PHONY: clean console attach detach sshuttle ssh dry-run list deploy deploy-infra deploy-ceph deploy-openstack watch openstack create log login
