---
id: aws-s3-file-management
alias: AWS S3 File Management
type: kit
is_base: false
version: 1
tags:
  - middleware
  - storage
  - aws
description: Complete AWS S3 integration patterns including presigned URLs, lifecycle policies, versioning, CORS, and secure file upload/download workflows
---

# AWS S3 File Management Kit

A comprehensive kit for managing file storage with AWS S3, including secure uploads, presigned URLs, lifecycle management, and production-ready patterns.

## End State

After applying this kit, the application will have:

**S3 Bucket Configuration:**
- S3 bucket with versioning enabled
- Server-side encryption (SSE-S3 or SSE-KMS)
- Public access blocked for security
- CORS configuration for web uploads
- Lifecycle policies for cost optimization
- Bucket policies for access control

**File Upload/Download:**
- Secure direct-to-S3 uploads using presigned URLs
- Multipart upload support for large files
- Progress tracking for uploads
- File validation (size, type, content)
- Automatic thumbnail generation for images
- Virus scanning integration

**Access Control:**
- IAM roles and policies for service access
- Presigned URLs with expiration
- CloudFront CDN integration for public assets
- Private file access with temporary URLs
- Signed cookies for directory access

**File Management:**
- File metadata storage (database or DynamoDB)
- File organization by user/tenant/category
- Soft delete with retention period
- Automatic cleanup of orphaned files
- Backup and restore procedures

## Implementation Principles

- **Security first**: Always encrypt files at rest, use presigned URLs for temporary access
- **Direct uploads**: Use presigned URLs for client-side uploads to reduce server load
- **Validation**: Validate file types, sizes, and content on both client and server
- **Lifecycle management**: Implement lifecycle policies to move old files to cheaper storage
- **Versioning**: Enable versioning for important files to enable recovery
- **CDN integration**: Use CloudFront for public assets to reduce latency and costs
- **Error handling**: Implement retry logic and proper error handling for S3 operations
- **Monitoring**: Track upload/download metrics, errors, and costs
- **Cost optimization**: Use appropriate storage classes (Standard, IA, Glacier)
- **Access patterns**: Design bucket structure based on access patterns

## Pattern 1: S3 Client Configuration

Set up S3 client with proper configuration:

**Node.js (AWS SDK v3):**
```javascript
// s3/client.js
import { S3Client } from '@aws-sdk/client-s3';
import { fromEnv } from '@aws-sdk/credential-providers';

const s3Client = new S3Client({
  region: process.env.AWS_REGION || 'us-east-1',
  credentials: fromEnv(),
  // For local development
  // endpoint: 'http://localhost:4566', // LocalStack
  // forcePathStyle: true,
});

export default s3Client;
```

**Python (boto3):**
```python
# s3/client.py
import boto3
from botocore.config import Config

s3_client = boto3.client(
    's3',
    region_name=os.environ.get('AWS_REGION', 'us-east-1'),
    config=Config(
        signature_version='s3v4',
        retries={'max_attempts': 3}
    )
)
```

**Go:**
```go
// s3/client.go
import (
    "github.com/aws/aws-sdk-go/aws"
    "github.com/aws/aws-sdk-go/aws/session"
    "github.com/aws/aws-sdk-go/service/s3"
)

func NewS3Client() *s3.S3 {
    sess := session.Must(session.NewSession(&aws.Config{
        Region: aws.String(os.Getenv("AWS_REGION")),
    }))
    return s3.New(sess)
}
```

## Pattern 2: Generate Presigned URLs for Upload

Create presigned URLs for secure client-side uploads:

**Node.js:**
```javascript
// s3/presigned-upload.js
import { PutObjectCommand, GetObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';

export async function generateUploadUrl(key, contentType, expiresIn = 3600) {
  const command = new PutObjectCommand({
    Bucket: process.env.S3_BUCKET_NAME,
    Key: key,
    ContentType: contentType,
    // Server-side encryption
    ServerSideEncryption: 'AES256',
    // Metadata
    Metadata: {
      'uploaded-by': '{{USER_ID}}',
      'uploaded-at': new Date().toISOString(),
    },
  });

  const url = await getSignedUrl(s3Client, command, { expiresIn });
  return url;
}

// Usage
const uploadUrl = await generateUploadUrl(
  `uploads/${userId}/${fileId}`,
  'image/jpeg',
  3600 // 1 hour
);
```

**Python:**
```python
# s3/presigned_upload.py
import boto3
from botocore.config import Config

def generate_upload_url(key, content_type, expires_in=3600):
    s3_client = boto3.client('s3', config=Config(signature_version='s3v4'))
    
    url = s3_client.generate_presigned_url(
        'put_object',
        Params={
            'Bucket': os.environ['S3_BUCKET_NAME'],
            'Key': key,
            'ContentType': content_type,
            'ServerSideEncryption': 'AES256',
        },
        ExpiresIn=expires_in
    )
    return url
```

## Pattern 3: Multipart Upload for Large Files

Handle large file uploads with multipart upload:

**Node.js:**
```javascript
// s3/multipart-upload.js
import { CreateMultipartUploadCommand, UploadPartCommand, CompleteMultipartUploadCommand } from '@aws-sdk/client-s3';

export async function initiateMultipartUpload(key, contentType) {
  const command = new CreateMultipartUploadCommand({
    Bucket: process.env.S3_BUCKET_NAME,
    Key: key,
    ContentType: contentType,
    ServerSideEncryption: 'AES256',
  });

  const response = await s3Client.send(command);
  return response.UploadId;
}

export async function generatePartUploadUrl(key, uploadId, partNumber, expiresIn = 3600) {
  const command = new UploadPartCommand({
    Bucket: process.env.S3_BUCKET_NAME,
    Key: key,
    UploadId: uploadId,
    PartNumber: partNumber,
  });

  const url = await getSignedUrl(s3Client, command, { expiresIn });
  return url;
}

export async function completeMultipartUpload(key, uploadId, parts) {
  const command = new CompleteMultipartUploadCommand({
    Bucket: process.env.S3_BUCKET_NAME,
    Key: key,
    UploadId: uploadId,
    MultipartUpload: {
      Parts: parts.map((part, index) => ({
        ETag: part.etag,
        PartNumber: index + 1,
      })),
    },
  });

  return await s3Client.send(command);
}
```

## Pattern 4: Generate Presigned URLs for Download

Create temporary download URLs:

**Node.js:**
```javascript
// s3/presigned-download.js
import { GetObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';

export async function generateDownloadUrl(key, expiresIn = 3600, filename = null) {
  const params = {
    Bucket: process.env.S3_BUCKET_NAME,
    Key: key,
  };

  // Add response headers for download
  if (filename) {
    params.ResponseContentDisposition = `attachment; filename="${filename}"`;
  }

  const command = new GetObjectCommand(params);
  const url = await getSignedUrl(s3Client, command, { expiresIn });
  return url;
}

// For streaming/download
export async function generateStreamUrl(key, expiresIn = 3600) {
  const command = new GetObjectCommand({
    Bucket: process.env.S3_BUCKET_NAME,
    Key: key,
    ResponseContentDisposition: 'inline',
  });

  const url = await getSignedUrl(s3Client, command, { expiresIn });
  return url;
}
```

## Pattern 5: File Upload with Validation

Server-side file validation and processing:

**Node.js:**
```javascript
// s3/upload-handler.js
import { PutObjectCommand } from '@aws-sdk/client-s3';
import { v4 as uuidv4 } from 'uuid';
import sharp from 'sharp'; // For image processing

const ALLOWED_TYPES = ['image/jpeg', 'image/png', 'image/webp'];
const MAX_FILE_SIZE = 10 * 1024 * 1024; // 10MB

export async function uploadFile(file, userId) {
  // Validate file type
  if (!ALLOWED_TYPES.includes(file.mimetype)) {
    throw new Error('Invalid file type');
  }

  // Validate file size
  if (file.size > MAX_FILE_SIZE) {
    throw new Error('File too large');
  }

  // Generate unique key
  const fileId = uuidv4();
  const extension = file.originalname.split('.').pop();
  const key = `uploads/${userId}/${fileId}.${extension}`;

  // Process image (resize, optimize)
  let processedBuffer = file.buffer;
  if (file.mimetype.startsWith('image/')) {
    processedBuffer = await sharp(file.buffer)
      .resize(1920, 1920, { fit: 'inside', withoutEnlargement: true })
      .jpeg({ quality: 85 })
      .toBuffer();
  }

  // Upload to S3
  const command = new PutObjectCommand({
    Bucket: process.env.S3_BUCKET_NAME,
    Key: key,
    Body: processedBuffer,
    ContentType: file.mimetype,
    ServerSideEncryption: 'AES256',
    Metadata: {
      'original-name': file.originalname,
      'uploaded-by': userId,
      'uploaded-at': new Date().toISOString(),
    },
  });

  await s3Client.send(command);

  // Generate thumbnail
  const thumbnailBuffer = await sharp(file.buffer)
    .resize(300, 300, { fit: 'inside' })
    .jpeg({ quality: 80 })
    .toBuffer();

  const thumbnailKey = `thumbnails/${userId}/${fileId}.jpg`;
  const thumbnailCommand = new PutObjectCommand({
    Bucket: process.env.S3_BUCKET_NAME,
    Key: thumbnailKey,
    Body: thumbnailBuffer,
    ContentType: 'image/jpeg',
    ServerSideEncryption: 'AES256',
  });

  await s3Client.send(thumbnailCommand);

  return {
    key,
    thumbnailKey,
    url: `https://${process.env.S3_BUCKET_NAME}.s3.amazonaws.com/${key}`,
    thumbnailUrl: `https://${process.env.S3_BUCKET_NAME}.s3.amazonaws.com/${thumbnailKey}`,
  };
}
```

## Pattern 6: Lifecycle Policies

Configure lifecycle policies for cost optimization:

**Terraform:**
```hcl
resource "aws_s3_bucket_lifecycle_configuration" "main" {
  bucket = aws_s3_bucket.main.id

  rule {
    id     = "transition-to-ia"
    status = "Enabled"

    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }
  }

  rule {
    id     = "transition-to-glacier"
    status = "Enabled"

    transition {
      days          = 90
      storage_class = "GLACIER"
    }
  }

  rule {
    id     = "delete-old-versions"
    status = "Enabled"

    noncurrent_version_expiration {
      noncurrent_days = 365
    }
  }

  rule {
    id     = "delete-incomplete-multipart"
    status = "Enabled"

    abort_incomplete_multipart_upload {
      days_after_initiation = 7
    }
  }
}
```

**AWS CLI:**
```json
{
  "Rules": [
    {
      "Id": "transition-to-ia",
      "Status": "Enabled",
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "STANDARD_IA"
        }
      ]
    },
    {
      "Id": "delete-old-files",
      "Status": "Enabled",
      "Expiration": {
        "Days": 365
      }
    }
  ]
}
```

## Pattern 7: CORS Configuration

Configure CORS for web uploads:

**Terraform:**
```hcl
resource "aws_s3_bucket_cors_configuration" "main" {
  bucket = aws_s3_bucket.main.id

  cors_rule {
    allowed_headers = ["*"]
    allowed_methods = ["GET", "PUT", "POST", "DELETE", "HEAD"]
    allowed_origins = var.cors_allowed_origins
    expose_headers  = ["ETag", "x-amz-server-side-encryption"]
    max_age_seconds = 3000
  }
}
```

**AWS CLI:**
```json
{
  "CORSRules": [
    {
      "AllowedHeaders": ["*"],
      "AllowedMethods": ["GET", "PUT", "POST", "DELETE"],
      "AllowedOrigins": ["https://example.com"],
      "ExposeHeaders": ["ETag"],
      "MaxAgeSeconds": 3000
    }
  ]
}
```

## Pattern 8: CloudFront Integration

Serve files through CloudFront CDN:

**Terraform:**
```hcl
resource "aws_cloudfront_distribution" "s3" {
  origin {
    domain_name = aws_s3_bucket.main.bucket_regional_domain_name
    origin_id   = "S3-${aws_s3_bucket.main.id}"

    s3_origin_config {
      origin_access_identity = aws_cloudfront_origin_access_identity.main.cloudfront_access_identity_path
    }
  }

  enabled             = true
  is_ipv6_enabled     = true
  default_root_object = "index.html"

  default_cache_behavior {
    allowed_methods  = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "S3-${aws_s3_bucket.main.id}"

    forwarded_values {
      query_string = false
      cookies {
        forward = "none"
      }
    }

    viewer_protocol_policy = "redirect-to-https"
    min_ttl                = 0
    default_ttl            = 3600
    max_ttl                = 86400
  }

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  viewer_certificate {
    cloudfront_default_certificate = true
  }
}
```

## Pattern 9: File Deletion and Cleanup

Implement soft delete with cleanup:

**Node.js:**
```javascript
// s3/file-deletion.js
import { DeleteObjectCommand, ListObjectsV2Command } from '@aws-sdk/client-s3';

export async function softDeleteFile(key) {
  // Move to deleted folder with timestamp
  const deletedKey = `deleted/${Date.now()}-${key}`;
  
  // Copy to deleted location
  await s3Client.send(new CopyObjectCommand({
    Bucket: process.env.S3_BUCKET_NAME,
    CopySource: `${process.env.S3_BUCKET_NAME}/${key}`,
    Key: deletedKey,
  }));

  // Delete original
  await s3Client.send(new DeleteObjectCommand({
    Bucket: process.env.S3_BUCKET_NAME,
    Key: key,
  }));

  return deletedKey;
}

export async function cleanupOldDeletedFiles(daysOld = 30) {
  const cutoffDate = Date.now() - (daysOld * 24 * 60 * 60 * 1000);
  
  const command = new ListObjectsV2Command({
    Bucket: process.env.S3_BUCKET_NAME,
    Prefix: 'deleted/',
  });

  const objects = await s3Client.send(command);
  
  for (const obj of objects.Contents || []) {
    const timestamp = parseInt(obj.Key.split('/')[1].split('-')[0]);
    if (timestamp < cutoffDate) {
      await s3Client.send(new DeleteObjectCommand({
        Bucket: process.env.S3_BUCKET_NAME,
        Key: obj.Key,
      }));
    }
  }
}
```

## Pattern 10: IAM Policy for S3 Access

Create IAM policy for application access:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::{{BUCKET_NAME}}/uploads/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket"
      ],
      "Resource": "arn:aws:s3:::{{BUCKET_NAME}}",
      "Condition": {
        "StringLike": {
          "s3:prefix": "uploads/*"
        }
      }
    }
  ]
}
```

## Best Practices

1. **Encryption**: Always enable server-side encryption (SSE-S3 or SSE-KMS)
2. **Versioning**: Enable versioning for important files to enable recovery
3. **Lifecycle policies**: Use lifecycle policies to move old files to cheaper storage
4. **Presigned URLs**: Use presigned URLs for temporary access instead of making files public
5. **Validation**: Validate file types, sizes, and content on both client and server
6. **Multipart uploads**: Use multipart uploads for files larger than 5MB
7. **CDN integration**: Use CloudFront for public assets to reduce latency
8. **Monitoring**: Track upload/download metrics, errors, and costs
9. **Access control**: Use IAM policies and bucket policies for fine-grained access control
10. **Cost optimization**: Use appropriate storage classes based on access patterns

## Common Pitfalls

- **Public buckets**: Never make entire buckets public, use presigned URLs
- **Missing encryption**: Always enable server-side encryption
- **No lifecycle policies**: Old files accumulate and increase costs
- **Large file uploads**: Use multipart uploads for files > 5MB
- **Missing validation**: Validate files on server even if validated on client
- **Hardcoded credentials**: Use IAM roles, never hardcode AWS credentials
- **No error handling**: Implement retry logic for transient failures
- **Missing monitoring**: Track S3 usage and costs

## Verification Criteria

After generation, verify:
- ✓ S3 bucket has versioning enabled
- ✓ Server-side encryption is configured
- ✓ Public access is blocked
- ✓ CORS is configured for web uploads
- ✓ Lifecycle policies are set up
- ✓ Presigned URLs expire after specified time
- ✓ Multipart upload works for large files
- ✓ File validation works (type, size, content)
- ✓ IAM policies follow least-privilege principle
- ✓ CloudFront distribution is configured (if using CDN)
- ✓ Monitoring and logging are set up
- ✓ Cost alerts are configured
