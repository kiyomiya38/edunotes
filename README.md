# Ansible (Prod-lean)
- This playbook assumes secrets are stored in **SSM Parameter Store**:
  - `/${project}/db/password`
  - `/${project}/redis/password`
  - `/${project}/minio/access_key`
  - `/${project}/minio/secret_key`
- Instances have IAM role with permission to read SSM parameters.
- No SSH (use **SSM Session Manager**).
