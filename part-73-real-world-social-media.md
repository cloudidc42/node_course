# Part 73: Real-World Social Media API
## ขั้นตอนที่ 721-730 จาก 1000

---

## Social Media API Design

สร้าง social media platform backend ด้วย Node.js รองรับ:
- User profiles
- Follow/Following system
- Post/Feed
- Real-time notifications
- Likes & Comments

---

## 1. User Profiles

```javascript
// models/User.js
const userSchema = new mongoose.Schema({
  username: {
    type: String,
    required: true,
    unique: true,
    lowercase: true,
    trim: true,
    match: /^[a-z0-9_]{3,30}$/
  },
  displayName: { type: String, required: true },
  email: { type: String, required: true, unique: true, lowercase: true },
  password: { type: String, select: false },
  avatar: String,
  coverImage: String,
  bio: { type: String, maxlength: 160 },
  website: String,
  location: String,
  
  stats: {
    postsCount: { type: Number, default: 0 },
    followersCount: { type: Number, default: 0 },
    followingCount: { type: Number, default: 0 }
  },
  
  privacy: {
    isPrivate: { type: Boolean, default: false },
    showEmail: { type: Boolean, default: false }
  },
  
  verified: { type: Boolean, default: false },
  active: { type: Boolean, default: true },
  
  lastSeen: Date
}, { timestamps: true });

userSchema.index({ username: 1 });
userSchema.index({ displayName: 'text', bio: 'text' });
```

```javascript
// controllers/profile.controller.js
exports.getProfile = async (req, res) => {
  const user = await User.findOne({
    username: req.params.username,
    active: true
  }).select('-password -email');

  if (!user) return res.status(404).json({ error: 'User not found' });

  const isFollowing = req.user
    ? await Follow.exists({ follower: req.user.id, following: user._id })
    : false;

  res.json({
    ...user.toObject(),
    isFollowing: !!isFollowing,
    isOwnProfile: req.user?.id === user._id.toString()
  });
};

exports.updateProfile = async (req, res) => {
  const { displayName, bio, website, location, privacy } = req.body;
  
  const user = await User.findByIdAndUpdate(
    req.user.id,
    { displayName, bio, website, location, privacy },
    { new: true, runValidators: true }
  );

  res.json(user);
};

exports.searchUsers = async (req, res) => {
  const { q, page = 1, limit = 20 } = req.query;
  
  if (!q || q.length < 2) {
    return res.json({ data: [] });
  }

  const users = await User.find({
    $or: [
      { username: { $regex: q, $options: 'i' } },
      { displayName: { $regex: q, $options: 'i' } }
    ],
    active: true
  })
  .select('username displayName avatar verified stats.followersCount')
  .sort({ 'stats.followersCount': -1 })
  .skip((page - 1) * limit)
  .limit(parseInt(limit));

  res.json({ data: users });
};
```

---

## 2. Follow/Following System

```javascript
// models/Follow.js
const followSchema = new mongoose.Schema({
  follower: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  following: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  status: { 
    type: String, 
    enum: ['active', 'pending'], // pending สำหรับ private accounts
    default: 'active'
  }
}, { timestamps: true });

followSchema.index({ follower: 1, following: 1 }, { unique: true });
followSchema.index({ following: 1 });
```

```javascript
// controllers/follow.controller.js
exports.follow = async (req, res) => {
  const targetUser = await User.findById(req.params.userId);
  if (!targetUser) return res.status(404).json({ error: 'User not found' });
  
  if (targetUser._id.toString() === req.user.id) {
    return res.status(400).json({ error: 'Cannot follow yourself' });
  }

  const existing = await Follow.findOne({
    follower: req.user.id,
    following: targetUser._id
  });

  if (existing) {
    return res.status(400).json({ error: 'Already following' });
  }

  const status = targetUser.privacy.isPrivate ? 'pending' : 'active';

  await Follow.create({
    follower: req.user.id,
    following: targetUser._id,
    status
  });

  if (status === 'active') {
    await User.findByIdAndUpdate(req.user.id, { $inc: { 'stats.followingCount': 1 } });
    await User.findByIdAndUpdate(targetUser._id, { $inc: { 'stats.followersCount': 1 } });
    
    // ส่ง notification
    await notificationService.create({
      recipient: targetUser._id,
      sender: req.user.id,
      type: 'follow',
      data: {}
    });
  }

  res.json({
    following: true,
    status,
    message: status === 'pending' ? 'Follow request sent' : 'Following'
  });
};

exports.unfollow = async (req, res) => {
  const follow = await Follow.findOneAndDelete({
    follower: req.user.id,
    following: req.params.userId
  });

  if (!follow) return res.status(404).json({ error: 'Not following' });

  if (follow.status === 'active') {
    await User.findByIdAndUpdate(req.user.id, { $inc: { 'stats.followingCount': -1 } });
    await User.findByIdAndUpdate(req.params.userId, { $inc: { 'stats.followersCount': -1 } });
  }

  res.json({ following: false });
};

exports.getFollowers = async (req, res) => {
  const user = await User.findOne({ username: req.params.username });
  if (!user) return res.status(404).json({ error: 'User not found' });

  const { page = 1, limit = 20 } = req.query;
  
  const follows = await Follow.find({ following: user._id, status: 'active' })
    .populate('follower', 'username displayName avatar verified')
    .sort('-createdAt')
    .skip((page - 1) * limit)
    .limit(parseInt(limit));

  res.json({
    data: follows.map(f => f.follower),
    total: user.stats.followersCount
  });
};
```

---

## 3. Posts

```javascript
// models/Post.js
const postSchema = new mongoose.Schema({
  author: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  content: { type: String, maxlength: 2000 },
  images: [{ url: String, alt: String, width: Number, height: Number }],
  
  type: {
    type: String,
    enum: ['post', 'repost', 'reply'],
    default: 'post'
  },
  
  originalPost: { type: mongoose.Schema.Types.ObjectId, ref: 'Post' }, // สำหรับ repost
  replyTo: { type: mongoose.Schema.Types.ObjectId, ref: 'Post' },      // สำหรับ reply
  
  hashtags: [String],
  mentions: [{ type: mongoose.Schema.Types.ObjectId, ref: 'User' }],
  
  stats: {
    likesCount: { type: Number, default: 0 },
    commentsCount: { type: Number, default: 0 },
    repostsCount: { type: Number, default: 0 },
    viewsCount: { type: Number, default: 0 }
  },
  
  privacy: {
    type: String,
    enum: ['public', 'followers', 'private'],
    default: 'public'
  },
  
  status: { type: String, enum: ['active', 'deleted'], default: 'active' }
}, { timestamps: true });

postSchema.index({ author: 1, createdAt: -1 });
postSchema.index({ hashtags: 1 });
postSchema.index({ content: 'text' });

// Extract hashtags และ mentions ก่อน save
postSchema.pre('save', function() {
  if (this.isModified('content')) {
    this.hashtags = (this.content.match(/#(\w+)/g) || [])
      .map(tag => tag.slice(1).toLowerCase());
    
    // mentions handle ใน service layer
  }
});
```

```javascript
// controllers/post.controller.js
exports.create = async (req, res) => {
  const { content, images, privacy, replyTo } = req.body;
  
  if (!content && (!images || images.length === 0)) {
    return res.status(400).json({ error: 'Post must have content or images' });
  }

  // Extract mentions จาก content
  const mentionUsernames = (content?.match(/@(\w+)/g) || [])
    .map(m => m.slice(1));
  
  const mentionedUsers = await User.find({
    username: { $in: mentionUsernames }
  }).select('_id username');

  const post = await Post.create({
    author: req.user.id,
    content,
    images,
    privacy: privacy || 'public',
    mentions: mentionedUsers.map(u => u._id),
    replyTo: replyTo || undefined
  });

  // อัพเดต post count
  await User.findByIdAndUpdate(req.user.id, { $inc: { 'stats.postsCount': 1 } });
  
  // ส่ง mention notifications
  for (const user of mentionedUsers) {
    if (user._id.toString() !== req.user.id) {
      await notificationService.create({
        recipient: user._id,
        sender: req.user.id,
        type: 'mention',
        data: { postId: post._id }
      });
    }
  }

  const populatedPost = await Post.findById(post._id)
    .populate('author', 'username displayName avatar verified');

  res.status(201).json(populatedPost);
};

exports.like = async (req, res) => {
  const post = await Post.findById(req.params.id);
  if (!post) return res.status(404).json({ error: 'Post not found' });

  const existingLike = await Like.findOne({
    user: req.user.id,
    post: post._id
  });

  if (existingLike) {
    await existingLike.deleteOne();
    await Post.findByIdAndUpdate(post._id, { $inc: { 'stats.likesCount': -1 } });
    return res.json({ liked: false, likesCount: post.stats.likesCount - 1 });
  }

  await Like.create({ user: req.user.id, post: post._id });
  await Post.findByIdAndUpdate(post._id, { $inc: { 'stats.likesCount': 1 } });

  if (post.author.toString() !== req.user.id) {
    await notificationService.create({
      recipient: post.author,
      sender: req.user.id,
      type: 'like',
      data: { postId: post._id }
    });
  }

  res.json({ liked: true, likesCount: post.stats.likesCount + 1 });
};
```

---

## 4. Feed Algorithm

```javascript
// services/feed.service.js
const Redis = require('ioredis');
const redis = new Redis(process.env.REDIS_URL);

class FeedService {
  // Fan-out on write approach
  async pushToFeeds(post) {
    // ดึงรายชื่อ followers
    const followers = await Follow.find({
      following: post.author,
      status: 'active'
    }).select('follower');

    // Push post ไปยัง feed ของ followers
    const pipeline = redis.pipeline();
    
    for (const follow of followers) {
      const feedKey = `feed:${follow.follower}`;
      // เก็บ post ID พร้อม score = timestamp
      pipeline.zadd(feedKey, post.createdAt.getTime(), post._id.toString());
      // จำกัด feed ไม่เกิน 1000 posts
      pipeline.zremrangebyrank(feedKey, 0, -1001);
    }

    await pipeline.exec();
  }

  // ดึง feed สำหรับ user
  async getFeed(userId, page = 1, limit = 20) {
    const feedKey = `feed:${userId}`;
    const skip = (page - 1) * limit;
    
    // ดึง post IDs จาก Redis (เรียงตาม timestamp descending)
    const postIds = await redis.zrevrange(feedKey, skip, skip + limit - 1);

    if (postIds.length === 0) {
      // Fallback: ดึงจาก database
      return this.getFeedFromDB(userId, page, limit);
    }

    // ดึง posts จาก database
    const posts = await Post.find({
      _id: { $in: postIds },
      status: 'active'
    })
    .populate('author', 'username displayName avatar verified')
    .populate('originalPost');

    // จัดเรียงตาม postIds order
    const sortedPosts = postIds
      .map(id => posts.find(p => p._id.toString() === id))
      .filter(Boolean);

    return sortedPosts;
  }

  async getFeedFromDB(userId, page, limit) {
    // ดึง following IDs
    const following = await Follow.find({
      follower: userId,
      status: 'active'
    }).select('following');
    
    const followingIds = following.map(f => f.following);
    followingIds.push(userId); // รวม own posts

    return Post.find({
      author: { $in: followingIds },
      privacy: { $in: ['public', 'followers'] },
      status: 'active'
    })
    .populate('author', 'username displayName avatar verified')
    .sort('-createdAt')
    .skip((page - 1) * limit)
    .limit(limit);
  }

  // Explore feed (สำหรับ non-authenticated หรือ discover)
  async getExploreFeed(userId, page = 1, limit = 20) {
    const skip = (page - 1) * limit;
    
    // ใช้ weighted algorithm
    const posts = await Post.aggregate([
      { $match: { status: 'active', privacy: 'public' } },
      { $addFields: {
        score: {
          $add: [
            { $multiply: ['$stats.likesCount', 2] },
            { $multiply: ['$stats.commentsCount', 3] },
            { $multiply: ['$stats.repostsCount', 2] },
            // Recency factor
            { $divide: [
              { $subtract: [new Date(), '$createdAt'] },
              -3600000  // ลด score ทุก 1 ชั่วโมง
            ]}
          ]
        }
      }},
      { $sort: { score: -1 } },
      { $skip: skip },
      { $limit: limit }
    ]);

    return Post.populate(posts, { path: 'author', select: 'username displayName avatar verified' });
  }
}

module.exports = new FeedService();
```

---

## 5. Real-time Notifications

```javascript
// services/notification.service.js
const sseManager = require('./sse-manager');
const Notification = require('../models/Notification');

class NotificationService {
  async create({ recipient, sender, type, data }) {
    const notification = await Notification.create({
      recipient,
      sender,
      type,
      data
    });

    // ส่งแบบ real-time ผ่าน SSE
    const populated = await Notification.findById(notification._id)
      .populate('sender', 'username displayName avatar');

    sseManager.sendToChannel(`user:${recipient}`, populated, {
      event: 'notification'
    });

    return notification;
  }

  async getNotifications(userId, page = 1, limit = 20) {
    const [notifications, unreadCount] = await Promise.all([
      Notification.find({ recipient: userId })
        .populate('sender', 'username displayName avatar')
        .sort('-createdAt')
        .skip((page - 1) * limit)
        .limit(limit),
      Notification.countDocuments({ recipient: userId, read: false })
    ]);

    return { notifications, unreadCount };
  }

  async markAsRead(userId, notificationIds) {
    await Notification.updateMany(
      { _id: { $in: notificationIds }, recipient: userId },
      { read: true, readAt: new Date() }
    );
  }

  async markAllAsRead(userId) {
    await Notification.updateMany(
      { recipient: userId, read: false },
      { read: true, readAt: new Date() }
    );
  }
}

module.exports = new NotificationService();
```

---

## 6. Hashtag System

```javascript
// services/hashtag.service.js

exports.getTrending = async (req, res) => {
  // ดึง trending hashtags (24 ชั่วโมงที่ผ่านมา)
  const since = new Date(Date.now() - 24 * 60 * 60 * 1000);
  
  const trending = await Post.aggregate([
    { $match: { createdAt: { $gte: since }, status: 'active' } },
    { $unwind: '$hashtags' },
    { $group: { 
      _id: '$hashtags',
      count: { $sum: 1 },
      posts: { $addToSet: '$_id' }
    }},
    { $sort: { count: -1 } },
    { $limit: 20 },
    { $project: { 
      hashtag: '$_id',
      count: 1,
      postsCount: { $size: '$posts' }
    }}
  ]);

  res.json({ trending });
};

exports.getHashtagFeed = async (req, res) => {
  const { tag } = req.params;
  const { page = 1, limit = 20 } = req.query;

  const posts = await Post.find({
    hashtags: tag.toLowerCase(),
    status: 'active',
    privacy: 'public'
  })
  .populate('author', 'username displayName avatar verified')
  .sort('-createdAt')
  .skip((page - 1) * limit)
  .limit(parseInt(limit));

  const total = await Post.countDocuments({
    hashtags: tag.toLowerCase(),
    status: 'active'
  });

  res.json({ posts, total, hashtag: tag });
};
```

---

## แบบฝึกหัด

สร้าง Social Media API ที่มี:
1. User profiles + Follow system
2. Posts (text + images)
3. Likes + Comments
4. Home feed (from following)
5. Explore feed (trending)
6. Hashtags
7. Real-time notifications (SSE)
8. Search

---

## สรุป

Social Media APIs ซับซ้อนมาก โดยเฉพาะ feed algorithm และ notification system Redis เป็นสิ่งจำเป็นสำหรับ caching feed data การใช้ fan-out on write เหมาะกับ users ที่มี followers ไม่มาก ส่วน fan-out on read เหมาะกับ influencers ที่มี followers มาก (hybrid approach ดีที่สุด)

> ขั้นตอนต่อไป: Part 74 - Advanced Security
