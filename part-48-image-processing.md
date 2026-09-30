# Part 48: Image Processing (การประมวลผลรูปภาพ)
## ขั้นตอนที่ 48-48 จาก 1000

---

## บทนำ

การประมวลผลรูปภาพเป็นส่วนสำคัญของ web application ทุกตัวที่มีการ upload ไฟล์ ในบทนี้เราจะใช้ library **sharp** ซึ่งเป็น high-performance image processing library สำหรับ Node.js

---

## 48.1 Sharp Library

### ติดตั้ง Sharp

```bash
npm install sharp multer express
npm install --save-dev @types/multer
```

### ทำไมถึงใช้ Sharp?

- **เร็วมาก** - ใช้ libvips library ที่เขียนด้วย C
- **รองรับหลาย format** - JPEG, PNG, WebP, AVIF, TIFF, GIF, SVG
- **Pipeline processing** - chain operations ได้
- **Memory efficient** - ไม่ต้อง load รูปทั้งหมดเข้า RAM
- **Streaming support** - รองรับ Node.js streams

### การใช้งานเบื้องต้น

```javascript
const sharp = require('sharp');

// แปลงไฟล์
async function basicProcessing() {
  // อ่านและ resize
  await sharp('input.jpg')
    .resize(800, 600)
    .toFile('output.jpg');
  
  // อ่านจาก buffer
  const outputBuffer = await sharp(inputBuffer)
    .resize(300)
    .toBuffer();
  
  // ดูข้อมูล metadata
  const metadata = await sharp('image.jpg').metadata();
  console.log(metadata);
}
```

---

## 48.2 Resize, Crop, Convert

### Resize Options

```javascript
const sharp = require('sharp');

/**
 * Resize strategies
 */
async function resizeExamples(inputPath) {
  // 1. Resize ตาม width, height ตาม ratio
  await sharp(inputPath)
    .resize({ width: 800 })
    .toFile('width-only.jpg');
  
  // 2. Resize ตาม height, width ตาม ratio
  await sharp(inputPath)
    .resize({ height: 600 })
    .toFile('height-only.jpg');
  
  // 3. Fit ภายใน box (ไม่ crop)
  await sharp(inputPath)
    .resize(800, 600, { fit: 'inside' })
    .toFile('fit-inside.jpg');
  
  // 4. Cover box (crop เพื่อให้เต็ม)
  await sharp(inputPath)
    .resize(800, 600, {
      fit: 'cover',
      position: 'center'  // หรือ 'top', 'bottom', 'left', 'right'
    })
    .toFile('cover.jpg');
  
  // 5. Contain ภายใน box (มี padding)
  await sharp(inputPath)
    .resize(800, 600, {
      fit: 'contain',
      background: { r: 255, g: 255, b: 255, alpha: 1 }
    })
    .toFile('contain.jpg');
  
  // 6. Fill box (stretch)
  await sharp(inputPath)
    .resize(800, 600, { fit: 'fill' })
    .toFile('fill.jpg');
  
  // 7. Resize แบบ scale up ห้าม
  await sharp(inputPath)
    .resize(800, 600, {
      fit: 'inside',
      withoutEnlargement: true
    })
    .toFile('no-enlarge.jpg');
}
```

### Crop Operations

```javascript
/**
 * Crop รูปภาพ
 */
async function cropExamples(inputPath) {
  // 1. Extract ส่วนที่ต้องการ (left, top, width, height)
  await sharp(inputPath)
    .extract({ left: 100, top: 50, width: 400, height: 300 })
    .toFile('cropped.jpg');
  
  // 2. Smart crop - crop ที่น่าสนใจที่สุด
  await sharp(inputPath)
    .resize(400, 300, {
      fit: 'cover',
      position: sharp.strategy.attention  // focus on interesting areas
    })
    .toFile('smart-crop.jpg');
  
  // 3. Entropy-based crop
  await sharp(inputPath)
    .resize(400, 300, {
      fit: 'cover',
      position: sharp.strategy.entropy
    })
    .toFile('entropy-crop.jpg');
  
  // 4. Trim whitespace borders
  await sharp(inputPath)
    .trim()
    .toFile('trimmed.jpg');
}
```

### Format Conversion

```javascript
/**
 * แปลง format
 */
async function convertFormats(inputPath) {
  // JPEG
  await sharp(inputPath)
    .jpeg({
      quality: 85,
      progressive: true,
      mozjpeg: true  // ใช้ mozjpeg encoder
    })
    .toFile('output.jpg');
  
  // PNG
  await sharp(inputPath)
    .png({
      compressionLevel: 9,
      adaptiveFiltering: true
    })
    .toFile('output.png');
  
  // WebP
  await sharp(inputPath)
    .webp({
      quality: 80,
      lossless: false,
      nearLossless: false
    })
    .toFile('output.webp');
  
  // AVIF (AV1 Image Format - ใหม่กว่า WebP)
  await sharp(inputPath)
    .avif({
      quality: 50,
      lossless: false,
      speed: 5  // 0=slow(best), 9=fast
    })
    .toFile('output.avif');
  
  // TIFF
  await sharp(inputPath)
    .tiff({
      compression: 'lzw'
    })
    .toFile('output.tiff');
}
```

---

## 48.3 WebP Conversion

WebP เป็น format ที่ Google พัฒนา ให้ไฟล์ขนาดเล็กกว่า JPEG/PNG แต่คุณภาพใกล้เคียงกัน

### WebP Conversion Service

```javascript
// src/services/imageService.js
const sharp = require('sharp');
const path = require('path');
const fs = require('fs').promises;

/**
 * แปลงรูปเป็น WebP พร้อม fallback
 */
const convertToWebP = async (inputPath, options = {}) => {
  const {
    quality = 80,
    lossless = false,
    outputDir = null
  } = options;
  
  const ext = path.extname(inputPath);
  const basename = path.basename(inputPath, ext);
  const dir = outputDir || path.dirname(inputPath);
  const outputPath = path.join(dir, `${basename}.webp`);
  
  await sharp(inputPath)
    .webp({ quality, lossless })
    .toFile(outputPath);
  
  // เปรียบเทียบขนาด
  const [originalStat, webpStat] = await Promise.all([
    fs.stat(inputPath),
    fs.stat(outputPath)
  ]);
  
  const compressionRatio = ((1 - webpStat.size / originalStat.size) * 100).toFixed(1);
  
  return {
    originalPath: inputPath,
    webpPath: outputPath,
    originalSize: originalStat.size,
    webpSize: webpStat.size,
    compressionPercent: compressionRatio,
    savings: `${compressionRatio}% smaller`
  };
};

/**
 * สร้างรูปหลาย format พร้อมกัน (responsive images)
 */
const createResponsiveImages = async (inputPath, options = {}) => {
  const {
    sizes = [320, 640, 768, 1024, 1280, 1920],
    formats = ['webp', 'jpeg'],
    quality = 80,
    outputDir
  } = options;
  
  const ext = path.extname(inputPath);
  const basename = path.basename(inputPath, ext);
  const dir = outputDir || path.dirname(inputPath);
  
  const results = [];
  
  for (const size of sizes) {
    for (const format of formats) {
      const outputPath = path.join(dir, `${basename}-${size}.${format}`);
      
      await sharp(inputPath)
        .resize(size, null, { fit: 'inside', withoutEnlargement: true })
        [format]({ quality })
        .toFile(outputPath);
      
      const stat = await fs.stat(outputPath);
      results.push({
        width: size,
        format,
        path: outputPath,
        size: stat.size
      });
    }
  }
  
  return results;
};

module.exports = { convertToWebP, createResponsiveImages };
```

### Middleware สำหรับ WebP Auto-conversion

```javascript
// src/middleware/webpConversion.js
const sharp = require('sharp');
const path = require('path');

/**
 * Middleware ที่แปลง uploaded image เป็น WebP อัตโนมัติ
 */
const autoConvertWebP = (options = {}) => {
  const { quality = 80, maxWidth = 1920, maxHeight = 1080 } = options;
  
  return async (req, res, next) => {
    if (!req.file && !req.files) return next();
    
    const files = req.files || [req.file];
    
    try {
      for (const file of files) {
        if (!file.mimetype.startsWith('image/')) continue;
        
        // แปลงเป็น WebP
        const webpBuffer = await sharp(file.buffer)
          .resize(maxWidth, maxHeight, {
            fit: 'inside',
            withoutEnlargement: true
          })
          .webp({ quality })
          .toBuffer();
        
        // อัพเดท file object
        file.buffer = webpBuffer;
        file.mimetype = 'image/webp';
        file.originalname = file.originalname.replace(
          /\.(jpg|jpeg|png|gif|bmp|tiff)$/i, 
          '.webp'
        );
        file.size = webpBuffer.length;
      }
      
      next();
    } catch (error) {
      next(error);
    }
  };
};

module.exports = { autoConvertWebP };
```

---

## 48.4 Thumbnail Generation

```javascript
// src/services/thumbnailService.js
const sharp = require('sharp');
const path = require('path');
const fs = require('fs').promises;

const THUMBNAIL_SIZES = {
  xs: { width: 50, height: 50 },
  sm: { width: 100, height: 100 },
  md: { width: 200, height: 200 },
  lg: { width: 400, height: 400 },
  banner: { width: 1200, height: 400 },
  og: { width: 1200, height: 630 }  // Open Graph image
};

/**
 * สร้าง thumbnail หลายขนาด
 */
const generateThumbnails = async (inputPath, options = {}) => {
  const {
    sizes = ['xs', 'sm', 'md'],
    quality = 80,
    format = 'webp',
    outputDir
  } = options;
  
  const ext = path.extname(inputPath);
  const basename = path.basename(inputPath, ext);
  const dir = outputDir || path.join(path.dirname(inputPath), 'thumbnails');
  
  // สร้างโฟลเดอร์ถ้ายังไม่มี
  await fs.mkdir(dir, { recursive: true });
  
  const thumbnails = {};
  
  for (const sizeName of sizes) {
    const size = THUMBNAIL_SIZES[sizeName];
    if (!size) continue;
    
    const outputPath = path.join(dir, `${basename}-${sizeName}.${format}`);
    
    await sharp(inputPath)
      .resize(size.width, size.height, {
        fit: 'cover',
        position: 'center'
      })
      [format]({ quality })
      .toFile(outputPath);
    
    thumbnails[sizeName] = {
      path: outputPath,
      width: size.width,
      height: size.height
    };
  }
  
  return thumbnails;
};

/**
 * สร้าง thumbnail แบบ circular (สำหรับ avatar)
 */
const generateCircularThumbnail = async (inputPath, size = 200, outputPath) => {
  const circleShape = Buffer.from(
    `<svg><circle cx="${size/2}" cy="${size/2}" r="${size/2}"/></svg>`
  );
  
  await sharp(inputPath)
    .resize(size, size, { fit: 'cover', position: 'center' })
    .composite([{
      input: circleShape,
      blend: 'dest-in'
    }])
    .png()  // PNG เพื่อรองรับ transparency
    .toFile(outputPath || inputPath.replace(/\.[^.]+$/, '-circle.png'));
};

/**
 * สร้าง placeholder image (สำหรับ lazy loading)
 */
const generatePlaceholder = async (inputPath) => {
  // สร้าง tiny image 10px แล้ว scale up (blur effect)
  const tinyBuffer = await sharp(inputPath)
    .resize(10, null, { fit: 'inside' })
    .jpeg({ quality: 20 })
    .toBuffer();
  
  return `data:image/jpeg;base64,${tinyBuffer.toString('base64')}`;
};

module.exports = {
  generateThumbnails,
  generateCircularThumbnail,
  generatePlaceholder,
  THUMBNAIL_SIZES
};
```

---

## 48.5 EXIF Data

EXIF (Exchangeable Image File Format) เก็บข้อมูล metadata ของรูปภาพ เช่น ขนาด, กล้อง, GPS location

```javascript
// src/utils/exifExtractor.js
const sharp = require('sharp');
const exif = require('exif');  // npm install exif

/**
 * อ่าน metadata ด้วย Sharp
 */
const getImageMetadata = async (imagePath) => {
  const metadata = await sharp(imagePath).metadata();
  
  return {
    format: metadata.format,
    width: metadata.width,
    height: metadata.height,
    channels: metadata.channels,
    hasAlpha: metadata.hasAlpha,
    colorSpace: metadata.space,
    density: metadata.density,  // DPI
    isProgressive: metadata.isProgressive,
    orientation: metadata.orientation,
    exif: metadata.exif ? parseExifBuffer(metadata.exif) : null,
    icc: metadata.icc ? 'present' : null,
    iptc: metadata.iptc ? 'present' : null,
    xmp: metadata.xmp ? 'present' : null,
    size: await getFileSize(imagePath)
  };
};

/**
 * Parse EXIF buffer
 */
const parseExifBuffer = (exifBuffer) => {
  try {
    const ExifParser = require('exif-parser');  // npm install exif-parser
    const parser = ExifParser.create(exifBuffer);
    const result = parser.parse();
    
    return {
      make: result.tags.Make,
      model: result.tags.Model,
      software: result.tags.Software,
      dateTimeOriginal: result.tags.DateTimeOriginal,
      dateTime: result.tags.DateTime,
      exposureTime: result.tags.ExposureTime,
      fNumber: result.tags.FNumber,
      iso: result.tags.ISO,
      focalLength: result.tags.FocalLength,
      flash: result.tags.Flash,
      orientation: result.tags.Orientation,
      gps: result.tags.GPSLatitude ? {
        latitude: result.tags.GPSLatitude,
        longitude: result.tags.GPSLongitude,
        altitude: result.tags.GPSAltitude
      } : null
    };
  } catch {
    return null;
  }
};

/**
 * ลบ EXIF data ออกจากรูป (privacy)
 */
const stripExifData = async (inputPath, outputPath) => {
  await sharp(inputPath)
    .rotate()  // auto-rotate ตาม EXIF orientation แล้วลบ EXIF
    .withMetadata(false)  // ลบ metadata ทั้งหมด
    .toFile(outputPath || inputPath);
};

/**
 * เพิ่ม watermark
 */
const addWatermark = async (inputPath, watermarkPath, outputPath, options = {}) => {
  const { 
    gravity = 'southeast',  // northeast, northwest, southeast, southwest, center
    opacity = 0.5,
    margin = 20
  } = options;
  
  const image = sharp(inputPath);
  const imageMetadata = await image.metadata();
  
  // Resize watermark
  const watermarkBuffer = await sharp(watermarkPath)
    .resize(
      Math.floor(imageMetadata.width * 0.2),  // 20% ของ width
      null,
      { fit: 'inside' }
    )
    .composite([{
      input: Buffer.from([0, 0, 0, Math.floor(opacity * 255)]),
      raw: { width: 1, height: 1, channels: 4 },
      tile: true,
      blend: 'dest-in'
    }])
    .toBuffer();
  
  await image
    .composite([{
      input: watermarkBuffer,
      gravity,
    }])
    .toFile(outputPath);
};

module.exports = { getImageMetadata, stripExifData, addWatermark };
```

---

## 48.6 Image Upload API

### Multer Setup

```javascript
// src/config/multer.js
const multer = require('multer');
const path = require('path');

// Memory storage - เก็บใน buffer ก่อน process
const memoryStorage = multer.memoryStorage();

// Disk storage - บันทึกตรง disk
const diskStorage = multer.diskStorage({
  destination: (req, file, cb) => {
    cb(null, 'uploads/temp/');
  },
  filename: (req, file, cb) => {
    const uniqueSuffix = `${Date.now()}-${Math.round(Math.random() * 1E9)}`;
    cb(null, `${file.fieldname}-${uniqueSuffix}${path.extname(file.originalname)}`);
  }
});

// File filter
const imageFilter = (req, file, cb) => {
  const allowedMimes = [
    'image/jpeg',
    'image/jpg',
    'image/png',
    'image/webp',
    'image/gif',
    'image/svg+xml'
  ];
  
  if (allowedMimes.includes(file.mimetype)) {
    cb(null, true);
  } else {
    cb(new Error(`ไม่รองรับประเภทไฟล์ ${file.mimetype}`), false);
  }
};

const upload = multer({
  storage: memoryStorage,
  fileFilter: imageFilter,
  limits: {
    fileSize: 10 * 1024 * 1024,  // 10 MB
    files: 5  // max 5 files
  }
});

module.exports = upload;
```

### Image Upload Controller

```javascript
// src/controllers/imageController.js
const sharp = require('sharp');
const path = require('path');
const fs = require('fs').promises;
const { generateThumbnails } = require('../services/thumbnailService');
const { convertToWebP } = require('../services/imageService');
const { getImageMetadata, stripExifData } = require('../utils/exifExtractor');

const UPLOAD_DIR = path.join(__dirname, '../../uploads');

/**
 * Upload และ process รูปภาพ
 */
const uploadImage = async (req, res) => {
  try {
    if (!req.file) {
      return res.status(400).json({
        success: false,
        message: 'กรุณาแนบไฟล์รูปภาพ'
      });
    }
    
    const { buffer, originalname, mimetype } = req.file;
    const { quality = 80, generateThumbs = true } = req.body;
    
    // สร้าง unique filename
    const timestamp = Date.now();
    const ext = path.extname(originalname);
    const basename = path.basename(originalname, ext);
    const safeBasename = basename.replace(/[^a-zA-Z0-9-_]/g, '_');
    const filename = `${safeBasename}_${timestamp}`;
    
    // สร้างโฟลเดอร์
    const uploadPath = path.join(UPLOAD_DIR, 'images');
    await fs.mkdir(uploadPath, { recursive: true });
    
    // Validate ว่าเป็นรูปจริง ๆ
    const metadata = await sharp(buffer).metadata();
    
    // ลบ EXIF (privacy protection)
    const processedBuffer = await sharp(buffer)
      .rotate()  // auto-rotate
      .withMetadata(false)
      .toBuffer();
    
    // บันทึกไฟล์ original (WebP)
    const originalPath = path.join(uploadPath, `${filename}.webp`);
    await sharp(processedBuffer)
      .webp({ quality: parseInt(quality) })
      .toFile(originalPath);
    
    let thumbnails = {};
    if (generateThumbs === 'true' || generateThumbs === true) {
      thumbnails = await generateThumbnails(originalPath, {
        sizes: ['xs', 'sm', 'md', 'lg'],
        format: 'webp',
        quality: 75
      });
    }
    
    // สร้าง base64 placeholder
    const placeholderBuffer = await sharp(processedBuffer)
      .resize(10, null, { fit: 'inside' })
      .jpeg({ quality: 20 })
      .toBuffer();
    
    const placeholder = `data:image/jpeg;base64,${placeholderBuffer.toString('base64')}`;
    
    const fileStat = await fs.stat(originalPath);
    
    res.json({
      success: true,
      message: 'อัพโหลดรูปภาพสำเร็จ',
      data: {
        filename: `${filename}.webp`,
        originalName: originalname,
        url: `/uploads/images/${filename}.webp`,
        size: fileStat.size,
        dimensions: {
          width: metadata.width,
          height: metadata.height
        },
        format: 'webp',
        placeholder,
        thumbnails: Object.entries(thumbnails).reduce((acc, [size, thumb]) => {
          acc[size] = {
            url: `/uploads/images/thumbnails/${path.basename(thumb.path)}`,
            width: thumb.width,
            height: thumb.height
          };
          return acc;
        }, {})
      }
    });
    
  } catch (error) {
    res.status(500).json({
      success: false,
      message: error.message
    });
  }
};

/**
 * Upload หลายรูป
 */
const uploadMultipleImages = async (req, res) => {
  try {
    if (!req.files || req.files.length === 0) {
      return res.status(400).json({
        success: false,
        message: 'กรุณาแนบไฟล์รูปภาพ'
      });
    }
    
    const results = [];
    
    for (const file of req.files) {
      const timestamp = Date.now() + Math.random();
      const filename = `img_${timestamp}.webp`;
      const outputPath = path.join(UPLOAD_DIR, 'images', filename);
      
      await sharp(file.buffer)
        .resize(1920, null, { fit: 'inside', withoutEnlargement: true })
        .webp({ quality: 80 })
        .toFile(outputPath);
      
      results.push({
        originalName: file.originalname,
        filename,
        url: `/uploads/images/${filename}`
      });
    }
    
    res.json({
      success: true,
      uploaded: results.length,
      data: results
    });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};

module.exports = { uploadImage, uploadMultipleImages };
```

---

## 48.7 Image Optimization Pipeline

```javascript
// src/services/imageOptimizer.js
const sharp = require('sharp');

/**
 * Full optimization pipeline
 */
const optimizeImage = async (input, options = {}) => {
  const {
    maxWidth = 1920,
    maxHeight = 1080,
    quality = 80,
    format = 'auto',  // auto, jpeg, png, webp, avif
    stripMetadata = true,
    progressive = true
  } = options;
  
  let pipeline = sharp(input);
  
  // Auto-rotate ตาม EXIF orientation
  pipeline = pipeline.rotate();
  
  // Strip metadata ถ้าต้องการ
  if (stripMetadata) {
    pipeline = pipeline.withMetadata(false);
  }
  
  // Resize ถ้าใหญ่เกิน
  pipeline = pipeline.resize(maxWidth, maxHeight, {
    fit: 'inside',
    withoutEnlargement: true
  });
  
  // เลือก output format
  const metadata = await sharp(input).metadata();
  let outputFormat = format;
  
  if (format === 'auto') {
    // เลือก format ที่เหมาะสมตาม content
    if (metadata.hasAlpha) {
      outputFormat = 'webp';  // WebP รองรับ transparency
    } else if (metadata.format === 'png') {
      outputFormat = 'webp';
    } else {
      outputFormat = 'jpeg';
    }
  }
  
  switch (outputFormat) {
    case 'jpeg':
    case 'jpg':
      pipeline = pipeline.jpeg({ quality, progressive, mozjpeg: true });
      break;
    case 'png':
      pipeline = pipeline.png({ compressionLevel: 9 });
      break;
    case 'webp':
      pipeline = pipeline.webp({ quality });
      break;
    case 'avif':
      pipeline = pipeline.avif({ quality });
      break;
  }
  
  return pipeline.toBuffer();
};

module.exports = { optimizeImage };
```

---

## แบบฝึกหัดที่ 48

### แบบฝึกหัดพื้นฐาน

**1. Image Upload API**

สร้าง API ที่:
- รับ upload รูปภาพ
- Resize ให้ max width 1200px
- แปลงเป็น WebP
- สร้าง thumbnail 3 ขนาด (100, 300, 600)
- Return URL ของทุกรูป

**2. EXIF Data Extractor**

สร้าง API ที่:
- รับ upload รูปภาพ
- Extract EXIF data
- แสดง GPS coordinates (ถ้ามี)
- ลบ EXIF ก่อน save (privacy)

### แบบฝึกหัดขั้นสูง

**3. Image CDN Simulation**

สร้าง image serving system ที่:
- รับ URL parameter สำหรับ resize: `/images/product-1.jpg?w=300&h=200&fit=cover`
- Cache ผลลัพธ์ใน Redis
- Serve รูปที่เหมาะสม (WebP ถ้า browser รองรับ)

**4. Watermark Service**

สร้าง API ที่:
- รับรูปและ watermark text หรือ logo
- ใส่ watermark ที่มุมขวาล่าง
- ปรับ opacity ได้
- รองรับ batch processing

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Sharp Library** - การใช้งาน high-performance image processing
2. **Resize/Crop** - strategies ต่าง ๆ (fit, cover, contain)
3. **WebP Conversion** - ลดขนาดไฟล์ 25-35%
4. **Thumbnail Generation** - หลายขนาดพร้อมกัน
5. **EXIF Data** - อ่านและลบ metadata
6. **Upload Pipeline** - process รูปก่อน save

**ถัดไป:** Part 49 - Background Jobs (งานเบื้องหลัง)
