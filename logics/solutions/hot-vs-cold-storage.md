# Hot vs Cold Storage Strategy for Chat Service

## Table of Contents
1. [Overview](#overview)
2. [Key Concepts](#key-concepts)
3. [Architecture Decision](#architecture-decision)
4. [S3 Storage Classes Explained](#s3-storage-classes-explained)
5. [Implementation Guide](#implementation-guide)
6. [Cost Analysis](#cost-analysis)
7. [When to Use Lifecycle Rules](#when-to-use-lifecycle-rules)
8. [Best Practices](#best-practices)

---

## Overview

This document explains the storage strategy for media files in the chat service, focusing on cost-effective storage using S3's automatic lifecycle transitions.

### The Problem
- Chat messages stored in Cassandra (text, metadata)
- Media files (images, videos, documents) can be large
- Old media is rarely accessed but must be retained
- Storing everything in "hot" storage is expensive

### The Solution
- Store text messages in Cassandra
- Store media files in S3 with automatic tier transitions
- Use S3 lifecycle rules to move old media to cheaper storage classes
- Achieve 60-80% cost savings on storage

---

## Key Concepts

### What is Hot vs Cold Storage?

**Hot Storage (STANDARD)**
- Fast access (milliseconds)
- Higher cost ($0.023/GB/month)
- For frequently accessed data
- Use for: Recent messages (0-30 days)

**Warm Storage (STANDARD_IA)**
- Fast access (milliseconds)
- Medium cost ($0.0125/GB/month)
- For infrequently accessed data
- Use for: Recent history (30-90 days)

**Cold Storage (GLACIER_INSTANT_RETRIEVAL)**
- Fast access (milliseconds)
- Low cost ($0.004/GB/month)
- For rarely accessed data
- Use for: Old messages (90-365 days)

**Frozen Storage (DEEP_ARCHIVE)**
- Slow access (12 hours)
- Very low cost ($0.00099/GB/month)
- For archives
- Use for: Very old messages (365+ days)

### Important Clarification: No Separate Buckets Needed!

**WRONG Approach:**
```typescript
// ❌ Don't do this - unnecessary complexity
const hotBucket = 's3://chat-media-hot';
const coldBucket = 's3://chat-media-cold';
```

**CORRECT Approach:**
```typescript
// ✅ Single bucket with different storage classes
const bucket = 's3://chat-media';

// Same bucket, different storage classes per object
await s3.putObject({
  Bucket: 'chat-media',
  Key: 'recent/image.jpg',
  StorageClass: 'STANDARD'  // Hot
});

await s3.putObject({
  Bucket: 'chat-media',
  Key: 'old/image.jpg',
  StorageClass: 'GLACIER_IR'  // Cold
});
```

---

## Architecture Decision

### What Gets Stored Where?

#### Cassandra Database
```typescript
// Store in Cassandra:
{
  messageId: "msg-123",
  chatId: "chat-456",
  userId: "user-789",
  text: "Check out this image!",
  timestamp: "2024-11-20T10:30:00Z",
  
  // Only references to media - not the actual files
  hasMedia: true,
  mediaReferences: [
    {
      mediaId: "media-001",
      storageKey: "chats/chat-456/2024/11/msg-123",
      storageTier: "hot",
      fileName: "screenshot.png",
      mimeType: "image/png",
      fileSize: 5242880,
      thumbnailKey: "thumbnails/media-001-thumb.webp"
    }
  ]
}
```

**Why Cassandra for Messages?**
- ✅ Fast text search and retrieval
- ✅ Excellent for time-series data (messages by timestamp)
- ✅ Horizontally scalable
- ✅ Low latency for message history

#### S3 Storage
```typescript
// Store in S3:
// - Actual binary files (images, videos, PDFs, documents)
// - Media files: 5MB image, 50MB video, 10MB PDF
```

**Why S3 for Media?**
- ✅ Cost-effective: $0.023/GB vs Cassandra's $0.25/GB
- ✅ Automatic lifecycle management
- ✅ Unlimited storage capacity
- ✅ Built-in redundancy and durability

### Cost Comparison

```
For 1TB of media files:

Cassandra: ~$250/month
S3 Standard: ~$23/month
S3 Glacier: ~$4/month

Savings: 84-98%
```

---

## S3 Storage Classes Explained

### How Automatic Transitions Work

```typescript
// Timeline for a file uploaded on January 1, 2025

Day 0 (Jan 1):    STANDARD          → $0.023/GB  → Upload
Day 30 (Jan 31):  STANDARD_IA       → $0.0125/GB → Auto transition
Day 90 (Apr 1):   GLACIER_IR        → $0.004/GB  → Auto transition  
Day 365 (Jan 1):  DEEP_ARCHIVE      → $0.00099/GB → Auto transition

All transitions happen automatically - you do nothing!
```

### Visual Timeline

```
Upload Date: January 1, 2025
File: document.pdf

┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Jan 1        Jan 31       Apr 1        Jan 1, 2026        │
│   │            │            │             │                 │
│   ▼            ▼            ▼             ▼                 │
│                                                             │
│ STANDARD → STANDARD_IA → GLACIER_IR → DEEP_ARCHIVE         │
│ (Hot)      (Warm)        (Cold)       (Frozen)             │
│                                                             │
│ $0.023/GB   $0.0125/GB   $0.004/GB    $0.00099/GB          │
│                                                             │
│  YOU DO:      S3 DOES:     S3 DOES:    S3 DOES:            │
│  Upload       Transition   Transition  Transition           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Storage Class Comparison Table

| Storage Class | Cost/GB/Month | Retrieval Time | Retrieval Cost | Use Case |
|--------------|---------------|----------------|----------------|----------|
| **STANDARD** | $0.023 | Milliseconds | Free | Active chats (0-30 days) |
| **STANDARD_IA** | $0.0125 | Milliseconds | $0.01/GB | Recent history (30-90 days) |
| **GLACIER_IR** | $0.004 | Milliseconds | $0.03/GB | Old messages (90-365 days) |
| **GLACIER_FLEXIBLE** | $0.0036 | 1-5 minutes | $0.01/GB | Archives (365+ days) |
| **DEEP_ARCHIVE** | $0.00099 | 12 hours | $0.02/GB | Long-term archives (1+ years) |

### Important Constraints

```typescript
const storageConstraints = {
  // Minimum storage durations
  STANDARD_IA: {
    minimumDays: 30,
    earlyDeletionCharge: 'Yes - charged for full 30 days'
  },
  GLACIER_IR: {
    minimumDays: 90,
    earlyDeletionCharge: 'Yes - charged for full 90 days'
  },
  DEEP_ARCHIVE: {
    minimumDays: 180,
    earlyDeletionCharge: 'Yes - charged for full 180 days'
  },
  
  // Minimum object size
  minimumBillableSize: '128 KB',
  note: 'Objects smaller than 128KB charged as 128KB'
};
```

---

## Implementation Guide

### 1. Configuration

```typescript
// filepath: src/config/storage.config.ts

export const storageConfig = {
  // Single bucket for all media
  bucket: process.env.S3_BUCKET || 'chat-media',
  region: process.env.AWS_REGION || 'us-east-1',
  
  // Storage classes
  storageClasses: {
    hot: 'STANDARD',                    // $0.023/GB
    warm: 'STANDARD_IA',                // $0.0125/GB
    cold: 'GLACIER_INSTANT_RETRIEVAL',  // $0.004/GB
    frozen: 'DEEP_ARCHIVE',             // $0.00099/GB
  },
  
  // Lifecycle transition rules
  lifecycleRules: [
    {
      id: 'auto-tier-chat-media',
      filter: { prefix: 'chats/' },
      transitions: [
        { days: 30, storageClass: 'STANDARD_IA' },
        { days: 90, storageClass: 'GLACIER_INSTANT_RETRIEVAL' },
        { days: 365, storageClass: 'DEEP_ARCHIVE' },
      ],
      // Optional: Delete after 7 years
      expiration: { days: 2555 }
    },
  ],
  
  // Thumbnail strategy
  thumbnails: {
    alwaysInHotStorage: true,
    maxSize: { width: 200, height: 200 },
    format: 'webp',
    prefix: 'thumbnails/',
  },
};
```

### 2. Media Storage Service

```typescript
// filepath: src/services/media-storage.service.ts

import { S3Client, PutObjectCommand, GetObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';
import { storageConfig } from '../config/storage.config';

export class MediaStorageService {
  private s3Client: S3Client;
  private bucket: string;

  constructor() {
    this.s3Client = new S3Client({ region: storageConfig.region });
    this.bucket = storageConfig.bucket;
  }

  /**
   * Upload media file to S3
   * Always uploads to STANDARD (hot) storage
   * S3 lifecycle rules handle automatic transitions
   */
  async uploadMedia(
    file: Express.Multer.File,
    metadata: {
      userId: string;
      chatId: string;
      messageId: string;
    }
  ): Promise<string> {
    const key = this.generateKey(metadata);
    
    await this.s3Client.send(
      new PutObjectCommand({
        Bucket: this.bucket,
        Key: key,
        Body: file.buffer,
        ContentType: file.mimetype,
        
        // Always upload as STANDARD (hot storage)
        StorageClass: 'STANDARD',
        
        // Metadata for tracking
        Metadata: {
          userId: metadata.userId,
          chatId: metadata.chatId,
          messageId: metadata.messageId,
          uploadedAt: new Date().toISOString(),
          originalFileName: file.originalname,
        },
        
        // Tags for lifecycle management
        Tagging: `tier=hot&chatId=${metadata.chatId}`,
      })
    );

    return key;
  }

  /**
   * Retrieve media from S3
   * Works transparently regardless of storage class
   */
  async retrieveMedia(key: string): Promise<Buffer> {
    const response = await this.s3Client.send(
      new GetObjectCommand({
        Bucket: this.bucket,
        Key: key,
        // S3 automatically handles retrieval from any storage class
      })
    );

    return Buffer.from(await response.Body.transformToByteArray());
  }

  /**
   * Get signed URL for client-side download
   * Avoids downloading through your server
   */
  async getSignedUrl(key: string, expiresIn: number = 3600): Promise<string> {
    const command = new GetObjectCommand({
      Bucket: this.bucket,
      Key: key,
    });

    return getSignedUrl(this.s3Client, command, { expiresIn });
  }

  /**
   * Generate S3 key with organized structure
   */
  private generateKey(metadata: {
    chatId: string;
    messageId: string;
  }): string {
    const date = new Date();
    const year = date.getFullYear();
    const month = String(date.getMonth() + 1).padStart(2, '0');
    
    // Structure: chats/{chatId}/{year}/{month}/{messageId}
    return `chats/${metadata.chatId}/${year}/${month}/${metadata.messageId}`;
  }

  /**
   * Upload thumbnail (always kept in hot storage)
   */
  async uploadThumbnail(
    thumbnailBuffer: Buffer,
    originalKey: string
  ): Promise<string> {
    const thumbnailKey = `thumbnails/${originalKey}-thumb.webp`;
    
    await this.s3Client.send(
      new PutObjectCommand({
        Bucket: this.bucket,
        Key: thumbnailKey,
        Body: thumbnailBuffer,
        ContentType: 'image/webp',
        StorageClass: 'STANDARD', // Always hot
        // Exclude from lifecycle rules
        Tagging: 'tier=hot&type=thumbnail&lifecycle=exclude',
      })
    );

    return thumbnailKey;
  }
}
```

### 3. Database Schema (Cassandra)

```typescript
// filepath: src/models/message.schema.ts

/**
 * Cassandra table for messages
 * Stores text and metadata, references to media files
 */
export const messageSchema = `
  CREATE TABLE IF NOT EXISTS messages (
    chat_id text,
    message_id text,
    user_id text,
    text text,
    timestamp timestamp,
    
    -- Media references (not actual files)
    has_media boolean,
    media_references list<frozen<media_ref>>,
    
    PRIMARY KEY ((chat_id), timestamp, message_id)
  ) WITH CLUSTERING ORDER BY (timestamp DESC)
    AND compaction = {'class': 'TimeWindowCompactionStrategy'};
`;

/**
 * User-defined type for media references
 */
export const mediaRefType = `
  CREATE TYPE IF NOT EXISTS media_ref (
    media_id text,
    storage_key text,         -- S3 key
    storage_tier text,        -- Current tier (for tracking)
    file_name text,
    mime_type text,
    file_size bigint,
    thumbnail_key text,       -- Small thumbnail always in hot storage
    uploaded_at timestamp
  );
`;

/**
 * Index for querying media
 */
export const mediaIndex = `
  CREATE INDEX IF NOT EXISTS idx_messages_has_media
  ON messages (has_media);
`;
```

### 4. Database Schema (MongoDB - for tracking)

```typescript
// filepath: src/models/media.model.ts

import { Schema, model } from 'mongoose';

/**
 * Optional: Track media metadata in MongoDB
 * Useful for analytics and lifecycle tracking
 */
const mediaSchema = new Schema({
  messageId: { type: String, required: true, index: true },
  chatId: { type: String, required: true, index: true },
  userId: { type: String, required: true, index: true },
  
  // S3 information
  storageKey: { type: String, required: true, unique: true },
  bucket: { type: String, default: 'chat-media' },
  
  // Current storage class (updated by webhooks/events)
  currentStorageClass: {
    type: String,
    enum: ['STANDARD', 'STANDARD_IA', 'GLACIER_IR', 'DEEP_ARCHIVE'],
    default: 'STANDARD',
    index: true
  },
  
  // File metadata
  fileName: String,
  mimeType: String,
  fileSize: Number,
  
  // Thumbnail
  thumbnailKey: String,
  
  // Access tracking
  uploadedAt: { type: Date, default: Date.now, index: true },
  lastAccessedAt: { type: Date, default: Date.now },
  accessCount: { type: Number, default: 0 },
  
  // Lifecycle tracking
  lastTransitionedAt: Date,
  transitionHistory: [{
    from: String,
    to: String,
    transitionedAt: Date
  }]
}, {
  timestamps: true
});

// Method to track access
mediaSchema.methods.trackAccess = function() {
  this.lastAccessedAt = new Date();
  this.accessCount += 1;
  return this.save();
};

// Index for finding media to transition
mediaSchema.index({ currentStorageClass: 1, uploadedAt: 1 });

export const Media = model('Media', mediaSchema);
```

### 5. Setup S3 Lifecycle Rules (One-Time Setup)

#### Option A: Using AWS SDK

```typescript
// filepath: src/scripts/setup-s3-lifecycle.ts

import { S3Client, PutBucketLifecycleConfigurationCommand } from '@aws-sdk/client-s3';
import { storageConfig } from '../config/storage.config';

/**
 * Setup S3 lifecycle rules
 * Run this ONCE during infrastructure setup
 */
async function setupLifecycleRules() {
  const s3Client = new S3Client({ region: storageConfig.region });

  console.log('Setting up S3 lifecycle rules...');

  await s3Client.send(
    new PutBucketLifecycleConfigurationCommand({
      Bucket: storageConfig.bucket,
      LifecycleConfiguration: {
        Rules: [
          {
            Id: 'auto-tier-chat-media',
            Status: 'Enabled',
            
            // Apply to all chat media
            Filter: {
              And: {
                Prefix: 'chats/',
                Tags: [
                  { Key: 'lifecycle', Value: 'auto' }
                ]
              }
            },
            
            // Automatic transitions
            Transitions: [
              {
                Days: 30,
                StorageClass: 'STANDARD_IA',
              },
              {
                Days: 90,
                StorageClass: 'GLACIER_INSTANT_RETRIEVAL',
              },
              {
                Days: 365,
                StorageClass: 'DEEP_ARCHIVE',
              },
            ],
            
            // Optional: Delete after 7 years
            Expiration: {
              Days: 2555, // 7 years
            },
          },
          
          // Keep thumbnails always in hot storage
          {
            Id: 'keep-thumbnails-hot',
            Status: 'Enabled',
            Filter: {
              Prefix: 'thumbnails/'
            },
            // No transitions - stay in STANDARD
          }
        ],
      },
    })
  );

  console.log('✅ Lifecycle rules configured successfully!');
  console.log('S3 will now automatically transition objects:');
  console.log('  - Day 30: STANDARD → STANDARD_IA');
  console.log('  - Day 90: STANDARD_IA → GLACIER_IR');
  console.log('  - Day 365: GLACIER_IR → DEEP_ARCHIVE');
  console.log('  - Day 2555: Delete');
}

// Run setup
setupLifecycleRules().catch(console.error);
```

#### Option B: Using Terraform

```hcl
# filepath: infrastructure/s3.tf

resource "aws_s3_bucket" "chat_media" {
  bucket = "chat-media-bucket"
  
  tags = {
    Environment = "production"
    Service     = "chat-service"
  }
}

resource "aws_s3_bucket_lifecycle_configuration" "chat_media_lifecycle" {
  bucket = aws_s3_bucket.chat_media.id

  rule {
    id     = "auto-tier-chat-media"
    status = "Enabled"

    filter {
      and {
        prefix = "chats/"
        
        tags = {
          lifecycle = "auto"
        }
      }
    }

    # Transition to warm storage after 30 days
    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }

    # Transition to cold storage after 90 days
    transition {
      days          = 90
      storage_class = "GLACIER_INSTANT_RETRIEVAL"
    }

    # Transition to frozen storage after 1 year
    transition {
      days          = 365
      storage_class = "DEEP_ARCHIVE"
    }

    # Delete after 7 years
    expiration {
      days = 2555
    }
  }

  # Keep thumbnails in hot storage
  rule {
    id     = "keep-thumbnails-hot"
    status = "Enabled"

    filter {
      prefix = "thumbnails/"
    }

    # No transitions - thumbnails stay hot
  }
}

# Enable versioning for backup
resource "aws_s3_bucket_versioning" "chat_media_versioning" {
  bucket = aws_s3_bucket.chat_media.id
  
  versioning_configuration {
    status = "Enabled"
  }
}

# Enable server-side encryption
resource "aws_s3_bucket_server_side_encryption_configuration" "chat_media_encryption" {
  bucket = aws_s3_bucket.chat_media.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}
```

### 6. Message Service Integration

```typescript
// filepath: src/services/message.service.ts

import { MediaStorageService } from './media-storage.service';
import { Media } from '../models/media.model';

export class MessageService {
  private mediaStorage: MediaStorageService;

  constructor() {
    this.mediaStorage = new MediaStorageService();
  }

  /**
   * Send message with media
   */
  async sendMessage(data: {
    chatId: string;
    userId: string;
    text: string;
    files?: Express.Multer.File[];
  }) {
    const messageId = this.generateMessageId();
    const mediaReferences = [];

    // Upload media files if present
    if (data.files && data.files.length > 0) {
      for (const file of data.files) {
        // Upload to S3
        const storageKey = await this.mediaStorage.uploadMedia(file, {
          userId: data.userId,
          chatId: data.chatId,
          messageId,
        });

        // Create thumbnail for images
        let thumbnailKey;
        if (file.mimetype.startsWith('image/')) {
          const thumbnail = await this.generateThumbnail(file);
          thumbnailKey = await this.mediaStorage.uploadThumbnail(
            thumbnail,
            storageKey
          );
        }

        // Track in MongoDB (optional)
        const media = await Media.create({
          messageId,
          chatId: data.chatId,
          userId: data.userId,
          storageKey,
          fileName: file.originalname,
          mimeType: file.mimetype,
          fileSize: file.size,
          thumbnailKey,
          currentStorageClass: 'STANDARD',
        });

        // Add to media references
        mediaReferences.push({
          mediaId: media._id.toString(),
          storageKey,
          storageTier: 'hot',
          fileName: file.originalname,
          mimeType: file.mimetype,
          fileSize: file.size,
          thumbnailKey,
        });
      }
    }

    // Save message to Cassandra
    await this.cassandraClient.execute(
      `INSERT INTO messages (
        chat_id, message_id, user_id, text, timestamp,
        has_media, media_references
      ) VALUES (?, ?, ?, ?, ?, ?, ?)`,
      [
        data.chatId,
        messageId,
        data.userId,
        data.text,
        new Date(),
        mediaReferences.length > 0,
        mediaReferences,
      ],
      { prepare: true }
    );

    return { messageId, mediaReferences };
  }

  /**
   * Get message with media URLs
   */
  async getMessage(chatId: string, messageId: string) {
    // Get message from Cassandra
    const result = await this.cassandraClient.execute(
      'SELECT * FROM messages WHERE chat_id = ? AND message_id = ?',
      [chatId, messageId],
      { prepare: true }
    );

    const message = result.rows[0];

    if (!message) {
      throw new Error('Message not found');
    }

    // Generate signed URLs for media
    if (message.has_media && message.media_references) {
      const mediaUrls = await Promise.all(
        message.media_references.map(async (ref) => {
          // Track access
          await Media.findOneAndUpdate(
            { storageKey: ref.storage_key },
            {
              $set: { lastAccessedAt: new Date() },
              $inc: { accessCount: 1 }
            }
          );

          // Generate signed URL (valid for 1 hour)
          const url = await this.mediaStorage.getSignedUrl(
            ref.storage_key,
            3600
          );

          return {
            ...ref,
            url,
            thumbnailUrl: ref.thumbnail_key
              ? await this.mediaStorage.getSignedUrl(ref.thumbnail_key, 3600)
              : null,
          };
        })
      );

      return {
        ...message,
        media: mediaUrls,
      };
    }

    return message;
  }

  /**
   * Get chat history with optimized media loading
   */
  async getChatHistory(chatId: string, limit: number = 50) {
    const result = await this.cassandraClient.execute(
      'SELECT * FROM messages WHERE chat_id = ? LIMIT ?',
      [chatId, limit],
      { prepare: true }
    );

    // For history, only return thumbnails
    // Full media loaded on-demand when user clicks
    return result.rows.map(msg => ({
      ...msg,
      mediaThumbnails: msg.media_references?.map(ref => ({
        thumbnailKey: ref.thumbnail_key,
        thumbnailUrl: ref.thumbnail_key
          ? this.mediaStorage.getSignedUrl(ref.thumbnail_key, 3600)
          : null,
        fileName: ref.file_name,
        mimeType: ref.mime_type,
      })),
    }));
  }

  private generateMessageId(): string {
    return `msg-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
  }

  private async generateThumbnail(file: Express.Multer.File): Promise<Buffer> {
    // Implementation using Sharp library
    const sharp = require('sharp');
    
    return sharp(file.buffer)
      .resize(200, 200, { fit: 'inside' })
      .webp({ quality: 80 })
      .toBuffer();
  }
}
```

### 7. Optional: Background Job for Manual Transitions

```typescript
// filepath: src/jobs/media-tiering.job.ts

/**
 * Optional background job for manual tier management
 * NOTE: Only needed if you want custom logic beyond S3 lifecycle rules
 */

import { CopyObjectCommand, DeleteObjectCommand, HeadObjectCommand } from '@aws-sdk/client-s3';
import { Media } from '../models/media.model';

export class MediaTieringJob {
  
  /**
   * Check and migrate media that doesn't match expected tier
   * Useful for debugging or custom migration logic
   */
  async auditAndMigrate() {
    const cutoffDate = new Date();
    cutoffDate.setDate(cutoffDate.getDate() - 90);

    // Find media that should be cold but isn't
    const mediaToCheck = await Media.find({
      uploadedAt: { $lt: cutoffDate },
      currentStorageClass: { $in: ['STANDARD', 'STANDARD_IA'] },
    });

    for (const media of mediaToCheck) {
      await this.checkAndUpdateStorageClass(media);
    }
  }

  /**
   * Check actual storage class in S3
   */
  private async checkAndUpdateStorageClass(media: any) {
    try {
      const headResponse = await this.s3Client.send(
        new HeadObjectCommand({
          Bucket: storageConfig.bucket,
          Key: media.storageKey,
        })
      );

      const actualStorageClass = headResponse.StorageClass || 'STANDARD';

      // Update database if changed
      if (actualStorageClass !== media.currentStorageClass) {
        media.currentStorageClass = actualStorageClass;
        media.lastTransitionedAt = new Date();
        media.transitionHistory.push({
          from: media.currentStorageClass,
          to: actualStorageClass,
          transitionedAt: new Date()
        });
        await media.save();

        console.log(`Updated ${media.storageKey}: ${actualStorageClass}`);
      }
    } catch (error) {
      console.error(`Error checking ${media.storageKey}:`, error);
    }
  }

  /**
   * Generate storage analytics report
   */
  async generateStorageReport() {
    const report = await Media.aggregate([
      {
        $group: {
          _id: '$currentStorageClass',
          count: { $sum: 1 },
          totalSize: { $sum: '$fileSize' },
          avgAccessCount: { $avg: '$accessCount' }
        }
      },
      {
        $project: {
          storageClass: '$_id',
          count: 1,
          totalSizeGB: { $divide: ['$totalSize', 1073741824] },
          avgAccessCount: { $round: ['$avgAccessCount', 2] }
        }
      }
    ]);

    console.log('Storage Distribution Report:');
    console.table(report);

    return report;
  }
}
```

---

## Cost Analysis

### Example: 10TB Storage Over 1 Year

#### Without Lifecycle Rules (All STANDARD)
```
Total Storage: 10,000 GB
Cost: 10,000 GB × $0.023/GB = $230/month
Annual Cost: $2,760/year
```

#### With Lifecycle Rules (Automatic Tiering)

```typescript
// Breakdown by tier (estimated distribution)
const storageDistribution = {
  STANDARD: {
    percentage: 20,        // Recent 30 days
    storageGB: 2000,
    costPerGB: 0.023,
    monthlyCost: 2000 * 0.023, // = $46
  },
  STANDARD_IA: {
    percentage: 20,        // 30-90 days
    storageGB: 2000,
    costPerGB: 0.0125,
    monthlyCost: 2000 * 0.0125, // = $25
  },
  GLACIER_IR: {
    percentage: 40,        // 90-365 days
    storageGB: 4000,
    costPerGB: 0.004,
    monthlyCost: 4000 * 0.004, // = $16
  },
  DEEP_ARCHIVE: {
    percentage: 20,        // 365+ days
    storageGB: 2000,
    costPerGB: 0.00099,
    monthlyCost: 2000 * 0.00099, // = $2
  }
};

// Total monthly cost
const totalMonthlyCost = 46 + 25 + 16 + 2; // = $89/month
const annualCost = 89 * 12; // = $1,068/year

// Savings
const savings = {
  monthly: 230 - 89, // = $141/month
  annual: 2760 - 1068, // = $1,692/year
  percentage: ((230 - 89) / 230 * 100).toFixed(1) // = 61.3%
};
```

**Annual Savings: $1,692 (61.3% reduction)**

### Retrieval Costs (Additional)

```typescript
// Typical retrieval patterns
const retrievalCosts = {
  STANDARD: {
    retrievalCostPerGB: 0,
    requestCost: '$0.0004 per 1000 requests'
  },
  STANDARD_IA: {
    retrievalCostPerGB: 0.01,
    requestCost: '$0.001 per 1000 requests',
    example: '100GB retrieved = $1 + request costs'
  },
  GLACIER_IR: {
    retrievalCostPerGB: 0.03,
    requestCost: '$0.01 per 1000 requests',
    example: '100GB retrieved = $3 + request costs'
  },
  DEEP_ARCHIVE: {
    retrievalCostPerGB: 0.02,
    requestCost: '$0.05 per 1000 requests',
    example: '100GB retrieved = $2 + request costs',
    note: 'Plus 12-hour wait time'
  }
};

// Typical monthly retrieval (estimated)
const monthlyRetrieval = {
  STANDARD_IA: '50 GB × $0.01 = $0.50',
  GLACIER_IR: '10 GB × $0.03 = $0.30',
  DEEP_ARCHIVE: '1 GB × $0.02 = $0.02',
  total: '$0.82/month'
};

// Total cost with retrieval
const totalWithRetrieval = 89 + 0.82; // = $89.82/month
// Still massive savings!
```

### Cost Comparison Table

| Scenario | Monthly Cost | Annual Cost | Savings |
|----------|-------------|-------------|---------|
| All STANDARD | $230 | $2,760 | - |
| With Lifecycle | $89 | $1,068 | 61.3% |
| With Retrieval | $90 | $1,080 | 60.9% |

---

## When to Use Lifecycle Rules

### ✅ USE Lifecycle Rules When:

1. **Access Frequency Drops Over Time**
   - Chat/messaging services (your case!)
   - Email attachments
   - Social media posts/stories
   - Security camera footage

2. **Large Storage Volume**
   - Total storage > 100GB
   - Potential savings > $50/month
   - Worth the setup and management

3. **Long Retention Periods**
   - Must keep data > 30 days
   - Compliance requirements (medical, legal)
   - Historical archives

4. **Predictable Access Patterns**
   - Can define when access drops
   - Clear lifecycle stages
   - Known retention policies

### ❌ DON'T USE Lifecycle Rules When:

1. **Always Accessed Data**
   ```typescript
   // Examples:
   - User profile pictures (shown on every login)
   - Company logos and branding
   - Product catalog images
   - App icons and UI assets
   - Active application assets
   ```

2. **Small Storage Volume**
   ```typescript
   const smallStorage = {
     totalSize: '50 GB',
     monthlyCost: 50 * 0.023, // = $1.15
     potentialSavings: '$0.50-0.70',
     worthIt: false, // Not worth the complexity
   };
   ```

3. **Short Retention**
   ```typescript
   // Files deleted before transitions occur
   const tempFiles = {
     retention: '7 days',
     firstTransition: '30 days',
     decision: 'Just use expiration policy, no transitions'
   };
   ```

4. **CDN-Cached Content**
   ```typescript
   const cdnAssets = {
     bucket: 'static-assets',
     cdn: 'CloudFront',
     s3Requests: '< 1% (99%+ hit CDN cache)',
     decision: 'Keep in STANDARD, CDN handles caching'
   };
   ```

### Decision Matrix

```typescript
// filepath: src/utils/storage-decision.helper.ts

export function shouldUseLifecycleRules(config: {
  totalStorageGB: number;
  accessPatternDecaysDays: number;
  retentionDays: number;
  currentMonthlyCost: number;
}) {
  // Rule 1: Too small to matter
  if (config.totalStorageGB < 100) {
    return {
      useLifecycle: false,
      reason: 'Storage volume too small (<100GB)',
      recommendation: 'Keep in STANDARD for simplicity'
    };
  }

  // Rule 2: Always accessed
  if (config.accessPatternDecaysDays === 0) {
    return {
      useLifecycle: false,
      reason: 'Content always accessed',
      recommendation: 'Keep in STANDARD, consider CDN'
    };
  }

  // Rule 3: Short retention
  if (config.retentionDays < 30) {
    return {
      useLifecycle: false,
      reason: 'Deleted before first transition',
      recommendation: 'Use expiration policy only'
    };
  }

  // Rule 4: Access pattern decays over time
  if (config.accessPatternDecaysDays >= 30) {
    const potentialSavings = this.calculateSavings(config);
    
    if (potentialSavings > 50) {
      return {
        useLifecycle: true,
        reason: `Significant savings: $${potentialSavings}/month`,
        estimatedSavings: {
          monthly: potentialSavings,
          annual: potentialSavings * 12,
          percentage: ((potentialSavings / config.currentMonthlyCost) * 100).toFixed(1) + '%'
        }
      };
    }
  }

  return {
    useLifecycle: false,
    reason: 'Savings not worth complexity',
    recommendation: 'Monitor and revisit if storage grows'
  };
}

// Helper function
function calculateSavings(config: any): number {
  // Simplified calculation
  const withLifecycle = 
    (config.totalStorageGB * 0.2 * 0.023) +  // 20% hot
    (config.totalStorageGB * 0.2 * 0.0125) + // 20% warm
    (config.totalStorageGB * 0.4 * 0.004) +  // 40% cold
    (config.totalStorageGB * 0.2 * 0.00099); // 20% frozen
  
  const withoutLifecycle = config.totalStorageGB * 0.023;
  
  return withoutLifecycle - withLifecycle;
}
```

### Service Type Comparison

| Service Type | Use Lifecycle? | Reason |
|-------------|----------------|--------|
| **Chat Media** | ✅ YES | Access drops 90%+ after 30 days |
| **Profile Pictures** | ❌ NO | Accessed on every login |
| **Database Backups** | ✅ YES | Rarely accessed after creation |
| **Product Images** | ❌ NO | Shown on every product view |
| **Security Footage** | ✅ YES | Only reviewed during incidents |
| **Static Website** | ❌ NO | Served constantly via CDN |
| **Email Attachments** | ✅ YES | Old emails rarely opened |
| **App Icons** | ❌ NO | Used in every UI render |
| **Log Files** | ✅ YES | Active analysis only recent logs |
| **Medical Records** | ✅ YES | Long retention, rare access |

---

## Best Practices

### 1. Storage Organization

```typescript
// Organize by date for lifecycle management
const keyStructure = {
  good: 'chats/{chatId}/{year}/{month}/{messageId}',
  // Easy to apply lifecycle rules by prefix
  
  bad: 'chats/{messageId}',
  // Harder to target specific time periods
};

// Use consistent prefixes
const prefixes = {
  activeChats: 'chats/',
  thumbnails: 'thumbnails/',
  profilePics: 'profiles/',
  tempFiles: 'temp/',
};
```

### 2. Tagging Strategy

```typescript
// Use tags for flexible lifecycle rules
await s3.putObject({
  Bucket: 'chat-media',
  Key: key,
  Body: file,
  Tagging: [
    'tier=hot',
    'type=chat-media',
    'lifecycle=auto',
    `chatId=${chatId}`,
    `uploadDate=${new Date().toISOString()}`
  ].join('&')
});

// Can create rules based on tags
const lifecycleRule = {
  Filter: {
    And: {
      Prefix: 'chats/',
      Tags: [
        { Key: 'lifecycle', Value: 'auto' }
      ]
    }
  }
};
```

### 3. Thumbnail Strategy

```typescript
/**
 * Always keep thumbnails in hot storage
 * They're small but frequently accessed
 */
const thumbnailStrategy = {
  maxSize: { width: 200, height: 200 },
  format: 'webp',  // Smaller file size
  quality: 80,
  
  storage: {
    class: 'STANDARD',  // Always hot
    prefix: 'thumbnails/',
    excludeFromLifecycle: true,
  },
  
  costImpact: {
    thumbnailSize: '20 KB average',
    per1000Thumbnails: '20 MB',
    monthlyCost: '20 MB × $0.023 = $0.0005',
    decision: 'Negligible cost, keep hot'
  }
};
```

### 4. Monitoring and Alerts

```typescript
// filepath: src/monitoring/storage-monitoring.ts

/**
 * Monitor storage distribution and costs
 */
export class StorageMonitoring {
  
  async getStorageMetrics() {
    // Get metrics from S3
    const metrics = await this.getS3Metrics();
    
    return {
      totalStorage: {
        sizeGB: metrics.totalBytes / 1073741824,
        costPerMonth: this.calculateCost(metrics)
      },
      
      byStorageClass: {
        STANDARD: {
          sizeGB: metrics.standard / 1073741824,
          percentage: (metrics.standard / metrics.totalBytes * 100).toFixed(1)
        },
        STANDARD_IA: {
          sizeGB: metrics.standardIA / 1073741824,
          percentage: (metrics.standardIA / metrics.totalBytes * 100).toFixed(1)
        },
        GLACIER_IR: {
          sizeGB: metrics.glacierIR / 1073741824,
          percentage: (metrics.glacierIR / metrics.totalBytes * 100).toFixed(1)
        },
        DEEP_ARCHIVE: {
          sizeGB: metrics.deepArchive / 1073741824,
          percentage: (metrics.deepArchive / metrics.totalBytes * 100).toFixed(1)
        }
      },
      
      projectedMonthlyCost: this.calculateCost(metrics),
      savingsFromLifecycle: this.calculateSavings(metrics)
    };
  }

  /**
   * Alert if storage distribution is unexpected
   */
  async checkStorageHealth() {
    const metrics = await this.getStorageMetrics();
    
    // Alert if too much in hot storage (>30%)
    if (metrics.byStorageClass.STANDARD.percentage > 30) {
      console.warn(`
        ⚠️ High hot storage usage: ${metrics.byStorageClass.STANDARD.percentage}%
        Expected: <30%
        Action: Check if lifecycle rules are working
      `);
    }
    
    // Alert if costs spike
    if (metrics.projectedMonthlyCost > this.expectedCost * 1.2) {
      console.warn(`
        ⚠️ Storage costs 20% above expected
        Projected: $${metrics.projectedMonthlyCost}
        Expected: $${this.expectedCost}
      `);
    }
  }
}
```

### 5. Testing Lifecycle Transitions

```typescript
// filepath: src/tests/lifecycle-transition.test.ts

/**
 * Test lifecycle transitions
 * NOTE: S3 lifecycle rules run once per day, so testing requires:
 * 1. Uploading objects with backdated timestamps (not possible)
 * 2. Waiting 30+ days (not practical)
 * 3. Manual transition for testing
 */

describe('Storage Lifecycle', () => {
  
  it('should upload to STANDARD storage', async () => {
    const key = await mediaStorage.uploadMedia(testFile, metadata);
    
    const headResponse = await s3.send(
      new HeadObjectCommand({ Bucket: bucket, Key: key })
    );
    
    expect(headResponse.StorageClass).toBeUndefined(); // STANDARD is default
  });

  it('should manually transition to GLACIER_IR', async () => {
    const key = await mediaStorage.uploadMedia(testFile, metadata);
    
    // Manually transition for testing
    await s3.send(
      new CopyObjectCommand({
        CopySource: `${bucket}/${key}`,
        Bucket: bucket,
        Key: key,
        StorageClass: 'GLACIER_INSTANT_RETRIEVAL',
        MetadataDirective: 'COPY'
      })
    );
    
    const headResponse = await s3.send(
      new HeadObjectCommand({ Bucket: bucket, Key: key })
    );
    
    expect(headResponse.StorageClass).toBe('GLACIER_IR');
  });

  it('should retrieve from GLACIER_IR successfully', async () => {
    // Upload and transition to cold storage
    const key = await mediaStorage.uploadMedia(testFile, metadata);
    await transitionToGlacier(key);
    
    // Retrieval should work transparently
    const retrieved = await mediaStorage.retrieveMedia(key);
    
    expect(retrieved).toBeDefined();
    expect(retrieved.length).toBeGreaterThan(0);
  });
});
```

### 6. Disaster Recovery

```typescript
// Enable versioning for backup
const versioningConfig = {
  enabled: true,
  
  // Keep deleted versions for 30 days
  lifecycleRule: {
    NoncurrentVersionExpiration: {
      NoncurrentDays: 30
    }
  }
};

// Enable cross-region replication for critical data
const replicationConfig = {
  enabled: true,
  destinationBucket: 'chat-media-backup-eu',
  destinationRegion: 'eu-west-1',
  
  // Only replicate hot storage
  filter: {
    Prefix: 'chats/',
    Tags: [
      { Key: 'tier', Value: 'hot' }
    ]
  }
};
```

### 7. Security Best Practices

```typescript
// Encrypt at rest
const encryptionConfig = {
  serverSide: 'AES256',  // or 'aws:kms' for more control
  enforced: true,
};

// Use signed URLs instead of public access
const accessControl = {
  publicAccess: false,
  signedUrlExpiry: 3600, // 1 hour
  
  // Different expiry for different tiers
  expiryByTier: {
    hot: 3600,      // 1 hour
    cold: 86400,    // 24 hours (slower access)
  }
};

// Implement bucket policies
const bucketPolicy = {
  Version: '2012-10-17',
  Statement: [
    {
      Sid: 'DenyUnencryptedObjectUploads',
      Effect: 'Deny',
      Principal: '*',
      Action: 's3:PutObject',
      Resource: 'arn:aws:s3:::chat-media/*',
      Condition: {
        StringNotEquals: {
          's3:x-amz-server-side-encryption': 'AES256'
        }
      }
    }
  ]
};
```

---

## Summary

### Key Takeaways

1. **Single Bucket, Multiple Storage Classes**
   - Don't create separate hot/cold buckets
   - Use storage classes within one bucket
   - S3 lifecycle rules handle transitions automatically

2. **Cassandra for Messages, S3 for Media**
   - Cassandra: Text, metadata, references
   - S3: Binary files (images, videos, documents)
   - Cost: 90%+ savings vs storing media in Cassandra

3. **Automatic Lifecycle Management**
   - Set rules once during setup
   - S3 handles everything automatically
   - No code changes needed for transitions

4. **Significant Cost Savings**
   - 60-80% reduction in storage costs
   - Example: $230/month → $90/month for 10TB
   - Annual savings: $1,500-2,000+

5. **Transparent Retrieval**
   - Same API regardless of storage class
   - Instant retrieval for STANDARD, STANDARD_IA, GLACIER_IR
   - Only DEEP_ARCHIVE has 12-hour delay

### Implementation Checklist

- [ ] Create S3 bucket
- [ ] Configure lifecycle rules
- [ ] Implement MediaStorageService
- [ ] Update message schema in Cassandra
- [ ] Implement thumbnail generation
- [ ] Set up monitoring and alerts
- [ ] Test upload and retrieval
- [ ] Document for team
- [ ] Monitor costs and optimize

### Next Steps

1. **Setup Infrastructure**
   ```bash
   # Run lifecycle setup script
   npm run setup:s3-lifecycle
   ```

2. **Deploy Services**
   ```bash
   # Deploy media storage service
   npm run deploy:media-service
   ```

3. **Monitor for 30 Days**
   ```bash
   # Check first transition after 30 days
   npm run storage:report
   ```

4. **Optimize**
   - Review storage distribution
   - Adjust lifecycle rules if needed
   - Monitor retrieval patterns and costs

---

## References

### AWS Documentation
- [S3 Storage Classes](https://aws.amazon.com/s3/storage-classes/)
- [S3 Lifecycle Configuration](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html)
- [S3 Pricing](https://aws.amazon.com/s3/pricing/)

### Internal Documentation
- `/docs/architecture/storage-architecture.md`
- `/docs/api/media-service-api.md`
- `/docs/operations/cost-optimization.md`

### Tools
- [AWS S3 Cost Calculator](https://calculator.aws/)
- [S3 Storage Lens](https://aws.amazon.com/s3/storage-lens/) - For analytics

---

**Last Updated:** December 20, 2025  
**Version:** 1.0  
**Author:** Chat Service Team