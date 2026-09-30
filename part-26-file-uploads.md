# Part 26: File Uploads

> ขั้นตอนที่ 26-30 จาก 1000 — การจัดการ File Upload ด้วย Multer, AWS S3 และ Sharp

---

## สารบัญ

1. [Multer Middleware](#multer-middleware)
2. [Local File Storage](#local-file-storage)
3. [Cloud Storage (AWS S3)](#cloud-storage-aws-s3)
4. [File Type Validation](#file-type-validation)
5. [Image Processing (Sharp)](#image-processing-sharp)
6. [Streaming Uploads](#streaming-uploads)
7. [Practical: Profile Picture Upload](#practical-profile-picture-upload)
8. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Multer Middleware

Multer เป็น middleware สำหรับจัดการ `multipart/form-data` (file uploads)

### ติดตั้ง

```bash
npm install multer
npm install sharp          # image processing
npm install @aws-sdk/client-s3 @aws-sdk/lib-storage  # AWS S3
npm install multer-s3      # S3 storage engine
npm install file-type      # ตรวจสอบ file type จาก buffer
```

### Multer พื้นฐาน

```javascript
// middleware/upload.js
const multer = require('multer');
const path = require('path');
const crypto = require('crypto');
const { AppError } = require('../utils/errors');

// กำหนด MIME types ที่อนุญาต
const ALLOWED_IMAGE_TYPES = ['image/jpeg', 'image/jpg', 'image/png', 'image/webp', 'image/gif'];
const ALLOWED_DOCUMENT_TYPES = ['application/pdf', 'application/msword', 'application/vnd.openxmlformats-officedocument.wordprocessingml.document'];
const MAX_FILE_SIZE = 5 * 1024 * 1024;  // 5 MB

// File filter
const imageFilter = (req, file, cb) => {
  if (ALLOWED_IMAGE_TYPES.includes(file.mimetype)) {
    cb(null, true);
  } else {
    cb(new AppError('อนุญาตเฉพาะไฟล์รูปภาพ (JPEG, PNG, WebP, GIF)', 400), false);
  }
};

// Disk storage (บันทึกไว้ใน disk)
const diskStorage = multer.diskStorage({
  destination: (req, file, cb) => {
    cb(null, path.join(__dirname, '../uploads/temp'));
  },
  filename: (req, file, cb) => {
    const uniqueSuffix = `${Date.now()}-${crypto.randomBytes(6).toString('hex')}`;
    const ext = path.extname(file.originalname).toLowerCase();
    cb(null, `${uniqueSuffix}${ext}`);
  },
});

// Memory storage (เก็บไว้ใน RAM — สำหรับ processing ก่อน)
const memoryStorage = multer.memoryStorage();

// Single image upload
exports.uploadSingleImage = multer({
  storage: memoryStorage,
  fileFilter: imageFilter,
  limits: {
    fileSize: MAX_FILE_SIZE,
    files: 1,
  },
}).single('image');

// Multiple images
exports.uploadImages = multer({
  storage: memoryStorage,
  fileFilter: imageFilter,
  limits: {
    fileSize: MAX_FILE_SIZE,
    files: 10,  // สูงสุด 10 ไฟล์
  },
}).array('images', 10);

// Mixed files (images + documents)
exports.uploadMixed = multer({
  storage: memoryStorage,
  fileFilter: (req, file, cb) => {
    const allowed = [...ALLOWED_IMAGE_TYPES, ...ALLOWED_DOCUMENT_TYPES];
    if (allowed.includes(file.mimetype)) {
      cb(null, true);
    } else {
      cb(new AppError('ประเภทไฟล์ไม่รองรับ', 400), false);
    }
  },
  limits: { fileSize: 10 * 1024 * 1024 },  // 10 MB สำหรับ documents
}).fields([
  { name: 'avatar', maxCount: 1 },
  { name: 'documents', maxCount: 5 },
]);

// Error handler สำหรับ multer
exports.handleUploadError = (err, req, res, next) => {
  if (err instanceof multer.MulterError) {
    if (err.code === 'LIMIT_FILE_SIZE') {
      return res.status(400).json({
        success: false,
        message: `ไฟล์ใหญ่เกินไป ขนาดสูงสุด ${MAX_FILE_SIZE / 1024 / 1024} MB`,
      });
    }
    if (err.code === 'LIMIT_FILE_COUNT') {
      return res.status(400).json({
        success: false,
        message: 'จำนวนไฟล์เกินที่กำหนด',
      });
    }
    return res.status(400).json({
      success: false,
      message: err.message,
    });
  }
  next(err);
};
```

---

## Local File Storage

### โครงสร้างโฟลเดอร์

```
uploads/
├── temp/          — temporary files (before processing)
├── images/
│   ├── avatars/
│   ├── posts/
│   └── products/
└── documents/
```

### Local Storage Service

```javascript
// services/localStorageService.js
const fs = require('fs').promises;
const path = require('path');
const sharp = require('sharp');

const UPLOADS_DIR = path.join(process.cwd(), 'uploads');

class LocalStorageService {
  constructor() {
    this.ensureDirectories();
  }
  
  async ensureDirectories() {
    const dirs = [
      path.join(UPLOADS_DIR, 'temp'),
      path.join(UPLOADS_DIR, 'images', 'avatars'),
      path.join(UPLOADS_DIR, 'images', 'posts'),
      path.join(UPLOADS_DIR, 'images', 'products'),
      path.join(UPLOADS_DIR, 'documents'),
    ];
    
    for (const dir of dirs) {
      await fs.mkdir(dir, { recursive: true });
    }
  }
  
  /**
   * บันทึก image พร้อม resize
   */
  async saveImage(buffer, options = {}) {
    const {
      folder = 'images',
      width = 1200,
      height = null,
      quality = 80,
      format = 'webp',
    } = options;
    
    const filename = `${Date.now()}-${crypto.randomBytes(8).toString('hex')}.${format}`;
    const filepath = path.join(UPLOADS_DIR, folder, filename);
    
    await sharp(buffer)
      .resize(width, height, {
        fit: 'inside',
        withoutEnlargement: true,
      })
      .toFormat(format, { quality })
      .toFile(filepath);
    
    return {
      filename,
      path: filepath,
      url: `/uploads/${folder}/${filename}`,
    };
  }
  
  /**
   * ลบไฟล์
   */
  async deleteFile(filepath) {
    try {
      await fs.unlink(filepath);
    } catch (error) {
      if (error.code !== 'ENOENT') throw error;
    }
  }
  
  /**
   * ดึงข้อมูลไฟล์
   */
  async getFileStats(filepath) {
    const stats = await fs.stat(filepath);
    return {
      size: stats.size,
      createdAt: stats.birthtime,
      modifiedAt: stats.mtime,
    };
  }
}

module.exports = new LocalStorageService();
```

### Serve Static Files

```javascript
// app.js
const express = require('express');
const path = require('path');

app.use('/uploads', express.static(path.join(__dirname, 'uploads'), {
  maxAge: '7d',  // cache 7 วัน
  etag: true,
  lastModified: true,
}));
```

---

## Cloud Storage (AWS S3)

### ตั้งค่า S3

```javascript
// config/s3.js
const { S3Client } = require('@aws-sdk/client-s3');

const s3Client = new S3Client({
  region: process.env.AWS_REGION || 'ap-southeast-1',
  credentials: {
    accessKeyId: process.env.AWS_ACCESS_KEY_ID,
    secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY,
  },
});

module.exports = s3Client;
```

### S3 Service

```javascript
// services/s3Service.js
const {
  PutObjectCommand,
  DeleteObjectCommand,
  GetObjectCommand,
} = require('@aws-sdk/client-s3');
const { getSignedUrl } = require('@aws-sdk/s3-request-presigner');
const { Upload } = require('@aws-sdk/lib-storage');
const s3Client = require('../config/s3');
const sharp = require('sharp');
const crypto = require('crypto');
const path = require('path');

const BUCKET_NAME = process.env.S3_BUCKET_NAME;
const CDN_URL = process.env.CLOUDFRONT_URL;  // ถ้าใช้ CloudFront

class S3Service {
  /**
   * Upload file ไป S3
   */
  async uploadFile(buffer, options = {}) {
    const {
      folder = 'uploads',
      contentType = 'application/octet-stream',
      filename,
      isPublic = true,
    } = options;
    
    const key = filename
      ? `${folder}/${filename}`
      : `${folder}/${Date.now()}-${crypto.randomBytes(8).toString('hex')}`;
    
    const command = new PutObjectCommand({
      Bucket: BUCKET_NAME,
      Key: key,
      Body: buffer,
      ContentType: contentType,
      ACL: isPublic ? 'public-read' : 'private',
      CacheControl: 'max-age=31536000',  // 1 ปี
    });
    
    await s3Client.send(command);
    
    const url = CDN_URL
      ? `${CDN_URL}/${key}`
      : `https://${BUCKET_NAME}.s3.${process.env.AWS_REGION}.amazonaws.com/${key}`;
    
    return { key, url };
  }
  
  /**
   * Upload image พร้อม resize
   */
  async uploadImage(buffer, options = {}) {
    const {
      folder = 'images',
      width = 1200,
      height = null,
      quality = 80,
      format = 'webp',
    } = options;
    
    const processedBuffer = await sharp(buffer)
      .resize(width, height, {
        fit: 'inside',
        withoutEnlargement: true,
      })
      .toFormat(format, { quality })
      .toBuffer();
    
    const filename = `${Date.now()}-${crypto.randomBytes(8).toString('hex')}.${format}`;
    
    return this.uploadFile(processedBuffer, {
      folder,
      contentType: `image/${format}`,
      filename,
    });
  }
  
  /**
   * Upload หลาย sizes (responsive images)
   */
  async uploadImageVariants(buffer, folder = 'images') {
    const sizes = [
      { name: 'thumbnail', width: 150, height: 150 },
      { name: 'small', width: 400 },
      { name: 'medium', width: 800 },
      { name: 'large', width: 1200 },
    ];
    
    const baseFilename = `${Date.now()}-${crypto.randomBytes(8).toString('hex')}`;
    
    const uploads = await Promise.all(
      sizes.map(({ name, width, height }) =>
        this.uploadImage(buffer, {
          folder: `${folder}/${name}`,
          width,
          height,
          filename: `${baseFilename}.webp`,
        })
      )
    );
    
    return sizes.reduce((acc, { name }, i) => {
      acc[name] = uploads[i].url;
      return acc;
    }, {});
  }
  
  /**
   * Multipart upload สำหรับไฟล์ใหญ่
   */
  async uploadLargeFile(stream, options = {}) {
    const { folder = 'uploads', contentType, filename } = options;
    
    const key = `${folder}/${filename || `${Date.now()}-${crypto.randomBytes(8).toString('hex')}`}`;
    
    const upload = new Upload({
      client: s3Client,
      params: {
        Bucket: BUCKET_NAME,
        Key: key,
        Body: stream,
        ContentType: contentType,
      },
      queueSize: 4,     // parallel uploads
      partSize: 5 * 1024 * 1024,  // 5 MB per part
    });
    
    upload.on('httpUploadProgress', (progress) => {
      console.log(`Upload progress: ${progress.loaded}/${progress.total}`);
    });
    
    await upload.done();
    
    return {
      key,
      url: `https://${BUCKET_NAME}.s3.${process.env.AWS_REGION}.amazonaws.com/${key}`,
    };
  }
  
  /**
   * สร้าง presigned URL (สำหรับ private files)
   */
  async getPresignedUrl(key, expiresIn = 3600) {
    const command = new GetObjectCommand({
      Bucket: BUCKET_NAME,
      Key: key,
    });
    
    return getSignedUrl(s3Client, command, { expiresIn });
  }
  
  /**
   * ลบไฟล์
   */
  async deleteFile(key) {
    const command = new DeleteObjectCommand({
      Bucket: BUCKET_NAME,
      Key: key,
    });
    
    await s3Client.send(command);
  }
}

module.exports = new S3Service();
```

---

## File Type Validation

### ตรวจสอบ Magic Bytes

MIME type จาก request ไม่น่าเชื่อถือ — ต้องตรวจสอบ file signature จาก buffer

```javascript
// utils/fileValidation.js
const fileType = require('file-type');

const ALLOWED_TYPES = {
  images: [
    { mime: 'image/jpeg', extensions: ['jpg', 'jpeg'] },
    { mime: 'image/png', extensions: ['png'] },
    { mime: 'image/webp', extensions: ['webp'] },
    { mime: 'image/gif', extensions: ['gif'] },
  ],
  documents: [
    { mime: 'application/pdf', extensions: ['pdf'] },
  ],
};

/**
 * ตรวจสอบ file type จาก buffer (ไม่ใช่จาก extension)
 */
exports.validateFileType = async (buffer, allowedMimes) => {
  const detected = await fileType.fromBuffer(buffer);
  
  if (!detected) {
    return {
      valid: false,
      message: 'ไม่สามารถระบุประเภทไฟล์ได้',
    };
  }
  
  if (!allowedMimes.includes(detected.mime)) {
    return {
      valid: false,
      message: `ประเภทไฟล์ ${detected.mime} ไม่รองรับ`,
      detected,
    };
  }
  
  return { valid: true, mime: detected.mime, ext: detected.ext };
};

/**
 * ตรวจสอบ image
 */
exports.validateImage = async (buffer) => {
  const allowedMimes = ALLOWED_TYPES.images.map((t) => t.mime);
  return exports.validateFileType(buffer, allowedMimes);
};

/**
 * Middleware ตรวจสอบ file type
 */
exports.validateImageMiddleware = async (req, res, next) => {
  if (!req.file) return next();
  
  const result = await exports.validateImage(req.file.buffer);
  
  if (!result.valid) {
    return res.status(400).json({
      success: false,
      message: result.message,
    });
  }
  
  req.file.detectedMime = result.mime;
  next();
};
```

---

## Image Processing (Sharp)

### Sharp พื้นฐาน

```javascript
// utils/imageProcessor.js
const sharp = require('sharp');

class ImageProcessor {
  /**
   * Resize image
   */
  async resize(buffer, width, height, options = {}) {
    return sharp(buffer)
      .resize(width, height, {
        fit: options.fit || 'cover',
        position: options.position || 'center',
        withoutEnlargement: true,
      })
      .toBuffer();
  }
  
  /**
   * แปลงเป็น WebP (ขนาดเล็กกว่า JPEG/PNG)
   */
  async toWebP(buffer, quality = 80) {
    return sharp(buffer)
      .webp({ quality, effort: 6 })
      .toBuffer();
  }
  
  /**
   * สร้าง thumbnail square
   */
  async createThumbnail(buffer, size = 200) {
    return sharp(buffer)
      .resize(size, size, { fit: 'cover', position: 'center' })
      .webp({ quality: 70 })
      .toBuffer();
  }
  
  /**
   * เพิ่ม watermark
   */
  async addWatermark(buffer, watermarkBuffer) {
    const image = sharp(buffer);
    const { width, height } = await image.metadata();
    
    const watermark = await sharp(watermarkBuffer)
      .resize(Math.floor(width * 0.3))  // 30% ของ image
      .composite([{ input: Buffer.from([0, 0, 0, 128]), raw: { width: 1, height: 1, channels: 4 } }])
      .toBuffer();
    
    return image
      .composite([
        {
          input: watermark,
          gravity: 'southeast',  // bottom-right
          blend: 'over',
        },
      ])
      .toBuffer();
  }
  
  /**
   * ดึงข้อมูล metadata
   */
  async getMetadata(buffer) {
    const metadata = await sharp(buffer).metadata();
    return {
      format: metadata.format,
      width: metadata.width,
      height: metadata.height,
      size: buffer.length,
      hasAlpha: metadata.hasAlpha,
    };
  }
  
  /**
   * Strip EXIF data (ลบข้อมูล GPS, camera model ฯลฯ)
   */
  async stripExif(buffer) {
    return sharp(buffer)
      .rotate()  // auto-rotate ตาม EXIF orientation
      .toBuffer();
  }
  
  /**
   * สร้าง responsive image set
   */
  async createImageSet(buffer, options = {}) {
    const { folder = 'uploads', baseKey } = options;
    const key = baseKey || `${Date.now()}-${Math.random().toString(36).slice(2)}`;
    
    const sizes = {
      thumb: { width: 150, height: 150, fit: 'cover' },
      small: { width: 400 },
      medium: { width: 800 },
      large: { width: 1400 },
      original: null,
    };
    
    const results = {};
    
    for (const [name, opts] of Object.entries(sizes)) {
      let processed = sharp(buffer).rotate();  // strip EXIF
      
      if (opts) {
        processed = processed.resize(opts.width, opts.height, {
          fit: opts.fit || 'inside',
          withoutEnlargement: true,
        });
      }
      
      const outputBuffer = await processed.webp({ quality: 85 }).toBuffer();
      results[name] = { buffer: outputBuffer, key: `${folder}/${name}/${key}.webp` };
    }
    
    return results;
  }
}

module.exports = new ImageProcessor();
```

---

## Streaming Uploads

### Stream โดยตรงไป S3

```javascript
// ไม่ต้องเก็บไฟล์ไว้ใน memory/disk ก่อน
const uploadStream = async (req, res) => {
  const { key } = req.params;
  
  const upload = new Upload({
    client: s3Client,
    params: {
      Bucket: BUCKET_NAME,
      Key: key,
      Body: req,  // req stream โดยตรง
      ContentType: req.headers['content-type'],
    },
    partSize: 5 * 1024 * 1024,
    queueSize: 3,
  });
  
  try {
    const result = await upload.done();
    res.json({ success: true, location: result.Location });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};
```

### Presigned Upload URL (Client อัปโหลดตรงไป S3)

```javascript
// สร้าง presigned URL ให้ client อัปโหลดตรง ไม่ผ่าน server
const { createPresignedPost } = require('@aws-sdk/s3-presigned-post');

exports.getUploadUrl = async (req, res) => {
  const { filename, contentType } = req.body;
  
  // ตรวจสอบ content type
  const allowedTypes = ['image/jpeg', 'image/png', 'image/webp'];
  if (!allowedTypes.includes(contentType)) {
    return res.status(400).json({ success: false, message: 'ประเภทไฟล์ไม่รองรับ' });
  }
  
  const key = `uploads/${req.user._id}/${Date.now()}-${filename}`;
  
  const { url, fields } = await createPresignedPost(s3Client, {
    Bucket: BUCKET_NAME,
    Key: key,
    Conditions: [
      ['content-length-range', 0, 5 * 1024 * 1024],  // สูงสุด 5 MB
      ['eq', '$Content-Type', contentType],
    ],
    Fields: {
      'Content-Type': contentType,
    },
    Expires: 300,  // 5 นาที
  });
  
  res.json({
    success: true,
    uploadUrl: url,
    fields,
    key,
  });
};

// Client-side ใช้งาน presigned POST
// const formData = new FormData();
// Object.entries(fields).forEach(([k, v]) => formData.append(k, v));
// formData.append('file', file);
// await fetch(uploadUrl, { method: 'POST', body: formData });
```

---

## Practical: Profile Picture Upload

### Upload Controller

```javascript
// controllers/uploadController.js
const { uploadSingleImage, handleUploadError } = require('../middleware/upload');
const s3Service = require('../services/s3Service');
const imageProcessor = require('../utils/imageProcessor');
const { validateImage } = require('../utils/fileValidation');
const User = require('../models/User');

exports.uploadAvatar = [
  // Multer middleware
  uploadSingleImage,
  
  // Error handler
  handleUploadError,
  
  // Controller
  async (req, res) => {
    try {
      if (!req.file) {
        return res.status(400).json({
          success: false,
          message: 'กรุณาเลือกไฟล์รูปภาพ',
        });
      }
      
      // ตรวจสอบ file type จาก buffer
      const validation = await validateImage(req.file.buffer);
      if (!validation.valid) {
        return res.status(400).json({
          success: false,
          message: validation.message,
        });
      }
      
      // Strip EXIF + resize สำหรับ avatar
      const processedBuffer = await imageProcessor.createThumbnail(
        req.file.buffer,
        200
      );
      
      // Upload ไป S3
      const { url, key } = await s3Service.uploadFile(processedBuffer, {
        folder: `avatars/${req.user._id}`,
        contentType: 'image/webp',
        filename: 'avatar.webp',
      });
      
      // ลบรูปเก่า
      if (req.user.avatarKey) {
        await s3Service.deleteFile(req.user.avatarKey).catch(console.error);
      }
      
      // อัปเดต user
      await User.findByIdAndUpdate(req.user._id, {
        avatar: url,
        avatarKey: key,
      });
      
      res.json({
        success: true,
        message: 'อัปโหลดรูปโปรไฟล์เรียบร้อยแล้ว',
        avatar: url,
      });
    } catch (error) {
      res.status(500).json({ success: false, message: error.message });
    }
  },
];

// Upload หลายรูปสำหรับ post
exports.uploadPostImages = [
  uploadImages,
  handleUploadError,
  
  async (req, res) => {
    try {
      if (!req.files || req.files.length === 0) {
        return res.status(400).json({ success: false, message: 'ไม่มีไฟล์' });
      }
      
      const uploadResults = await Promise.all(
        req.files.map(async (file) => {
          // สร้าง image variants
          const imageSet = await imageProcessor.createImageSet(file.buffer, {
            folder: `posts/${req.user._id}`,
          });
          
          const urls = {};
          for (const [size, { buffer, key }] of Object.entries(imageSet)) {
            const result = await s3Service.uploadFile(buffer, {
              folder: key.split('/').slice(0, -1).join('/'),
              contentType: 'image/webp',
              filename: key.split('/').pop(),
            });
            urls[size] = result.url;
          }
          
          return urls;
        })
      );
      
      res.json({
        success: true,
        images: uploadResults,
      });
    } catch (error) {
      res.status(500).json({ success: false, message: error.message });
    }
  },
];
```

### Routes

```javascript
// routes/upload.js
const express = require('express');
const router = express.Router();
const { authenticate } = require('../middleware/auth');
const uploadController = require('../controllers/uploadController');

router.post('/avatar', authenticate, ...uploadController.uploadAvatar);
router.post('/images', authenticate, ...uploadController.uploadPostImages);
router.post('/presigned', authenticate, uploadController.getUploadUrl);

module.exports = router;
```

---

## แบบฝึกหัด

### Exercise 1: Document Upload

1. รับไฟล์ PDF (สูงสุด 10 MB)
2. ตรวจสอบ magic bytes
3. สร้าง thumbnail จาก PDF หน้าแรก
4. เก็บใน S3 พร้อม metadata

### Exercise 2: Bulk Image Upload

1. รับไฟล์สูงสุด 20 ไฟล์พร้อมกัน
2. Process แบบ concurrent (Promise.all)
3. สร้าง WebP + thumbnail ทุกไฟล์
4. Return array ของ URLs

### Exercise 3: Image CDN

1. ตั้งค่า CloudFront CDN
2. สร้าง URL ที่มี transform parameters (resize on-the-fly)
3. Cache control headers
4. Signed URLs สำหรับ private images

---

## สรุป

- **Multer** middleware สำหรับ multipart form data
- **Magic bytes** validation ที่ปลอดภัยกว่า MIME type
- **Sharp** สำหรับ image processing ที่รวดเร็ว
- **AWS S3** สำหรับ cloud storage
- **Presigned URLs** ให้ client อัปโหลดตรงได้

> **บทถัดไป:** Part 27 — Email Sending
