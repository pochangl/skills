---
name: aws-credentials
description: "Use when: loading AWS Secrets Manager credentials with boto3; extending get_secret() pattern; managing region-specific AWS resources; setting up environment-aware credential hierarchy; implementing credential caching or error handling."
---

# AWS Credentials Loading with Boto3

Based on [server/credentials.py](../../../server/credentials.py), this skill provides patterns and best practices for loading AWS credentials and secrets in Django applications using boto3.

## Current Implementation

Your existing `get_secret()` function retrieves secrets from AWS Secrets Manager:

```python
# server/credentials.py
import boto3
import json
from botocore.exceptions import ClientError

def get_secret():
    secret_name = "sealchat"
    region_name = "ap-northeast-1"

    session = boto3.session.Session()
    client = session.client(
        service_name='secretsmanager',
        region_name=region_name
    )

    try:
        get_secret_value_response = client.get_secret_value(
            SecretId=secret_name
        )
    except ClientError as e:
        raise e

    secret = json.loads(get_secret_value_response['SecretString'])
    return secret
```

## Best Practices

1. **Use IAM roles in production** — avoid static access keys when possible
2. **Enable secret rotation** — configure automatic rotation in Secrets Manager
3. **Cache secrets with TTL** — balance freshness with API call costs
4. **Log without exposing secrets** — never log credential values
5. **Handle credential expiration gracefully** — implement clear_cache() on errors
6. **Use Secrets Manager for all sensitive data** — not just database passwords
7. **Organize secrets hierarchically** — e.g., `sealchat/database`, `sealchat/api-keys`
8. **Test with Moto** — mock Secrets Manager in unit tests
9. **Monitor access logs** — CloudTrail logs all Secrets Manager API calls
10. **Implement retry logic** — handle transient failures gracefully

## Common Issues

**ResourceNotFoundException:**
- Secret doesn't exist in specified region
- Check secret name and region match your resource

**DecryptionFailure:**
- KMS key used to encrypt secret is not accessible
- Verify IAM role has `kms:Decrypt` permission

**InvalidParameterException:**
- Invalid secret name format
- Secret name must be 1-512 characters

**Slow secret retrieval:**
- First call includes metadata service latency
- Use caching with appropriate TTL
- Consider pre-loading secrets on application startup

## Dependencies

Ensure boto3 is in requirements.txt:
```
boto3>=1.26.0
botocore>=1.29.0
```

For testing:
```
moto>=4.1.0  # Mock AWS services
pytest>=7.0
```
