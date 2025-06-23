# Finetune act
az ml job create --file .\aml\job_pi0.yaml --resource-group robotics-ch-north-secure --workspace-name robotics-ch-north-secure

# Finetune pi0 fast
az ml job create --file .\aml\job_pi0.yaml --resource-group robotics-ch-north-secure --workspace-name robotics-ch-north-secure
