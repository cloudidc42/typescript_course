# Part 58: Real-world Project - Social Media API (โปรเจกต์จริง: Social Media API)

## บทนำ

ในบทนี้เราจะสร้าง Social Media API ที่สมบูรณ์คล้ายกับ Twitter/Instagram ด้วย TypeScript, Node.js, Express, PostgreSQL และ Redis สำหรับ real-time features

## โครงสร้างโปรเจกต์ (Project Structure)

```
social-media-api/
├── src/
│   ├── config/
│   │   ├── database.ts
│   │   ├── redis.ts
│   │   └── websocket.ts
│   ├── types/
│   │   ├── models.ts
│   │   ├── requests.ts
│   │   ├── responses.ts
│   │   └── socket.ts
│   ├── models/
│   │   ├── User.ts
│   │   ├── Post.ts
│   │   ├── Comment.ts
│   │   ├── Like.ts
│   │   ├── Follow.ts
│   │   ├── Notification.ts
│   │   ├── Message.ts
│   │   └── Conversation.ts
│   ├── services/
│   │   ├── AuthService.ts
│   │   ├── UserService.ts
│   │   ├── PostService.ts
│   │   ├── CommentService.ts
│   │   ├── LikeService.ts
│   │   ├── FollowService.ts
│   │   ├── FeedService.ts
│   │   ├── NotificationService.ts
│   │   ├── MessageService.ts
│   │   └── SearchService.ts
│   ├── controllers/
│   │   ├── AuthController.ts
│   │   ├── UserController.ts
│   │   ├── PostController.ts
│   │   ├── CommentController.ts
│   │   ├── FeedController.ts
│   │   └── MessageController.ts
│   ├── middlewares/
│   │   ├── auth.ts
│   │   ├── upload.ts
│   │   └── rateLimit.ts
│   ├── routes/
│   │   ├── auth.ts
│   │   ├── users.ts
│   │   ├── posts.ts
│   │   ├── comments.ts
│   │   ├── feed.ts
│   │   └── messages.ts
│   ├── websocket/
│   │   ├── handlers/
│   │   │   ├── messageHandler.ts
│   │   │   └── notificationHandler.ts
│   │   └── index.ts
│   └── index.ts
├── prisma/
│   └── schema.prisma
├── package.json
└── tsconfig.json
```

---

## Core Types

### src/types/models.ts

```typescript
// ========================
// User Types
// ========================
export interface User {
  id: string;
  username: string;
  email: string;
  passwordHash: string;
  displayName: string;
  bio?: string;
  avatar?: string;
  coverImage?: string;
  website?: string;
  location?: string;
  dateOfBirth?: Date;
  isVerified: boolean;
  isPrivate: boolean;
  isActive: boolean;
  role: UserRole;
  
  // Stats
  followersCount: number;
  followingCount: number;
  postsCount: number;
  
  // Settings
  notificationSettings: NotificationSettings;
  privacySettings: PrivacySettings;
  
  createdAt: Date;
  updatedAt: Date;
}

export enum UserRole {
  USER = "USER",
  MODERATOR = "MODERATOR",
  ADMIN = "ADMIN"
}

export interface NotificationSettings {
  likes: boolean;
  comments: boolean;
  follows: boolean;
  mentions: boolean;
  messages: boolean;
  emailNotifications: boolean;
  pushNotifications: boolean;
}

export interface PrivacySettings {
  showEmail: boolean;
  showBirthday: boolean;
  showLocation: boolean;
  allowTagging: boolean;
  allowDirectMessages: "everyone" | "following" | "none";
}

// ========================
// Post Types
// ========================
export interface Post {
  id: string;
  authorId: string;
  author?: User;
  
  content?: string;
  media: PostMedia[];
  
  type: PostType;
  visibility: PostVisibility;
  
  // Quoted/Reposted post
  originalPostId?: string;
  originalPost?: Post;
  
  // Reply to
  replyToId?: string;
  replyTo?: Post;
  
  // Stats
  likesCount: number;
  commentsCount: number;
  repostsCount: number;
  viewsCount: number;
  
  // Metadata
  hashtags: string[];
  mentions: string[];
  location?: PostLocation;
  
  isEdited: boolean;
  isPinned: boolean;
  isSensitive: boolean;
  
  editHistory?: PostEdit[];
  
  createdAt: Date;
  updatedAt: Date;
}

export enum PostType {
  POST = "POST",
  REPLY = "REPLY",
  REPOST = "REPOST",
  QUOTE = "QUOTE"
}

export enum PostVisibility {
  PUBLIC = "PUBLIC",
  FOLLOWERS_ONLY = "FOLLOWERS_ONLY",
  MENTIONED_ONLY = "MENTIONED_ONLY",
  PRIVATE = "PRIVATE"
}

export interface PostMedia {
  id: string;
  postId: string;
  url: string;
  type: "image" | "video" | "gif";
  width?: number;
  height?: number;
  duration?: number; // seconds, for video
  thumbnail?: string;
  altText?: string;
  size: number; // bytes
  mimeType: string;
  sortOrder: number;
}

export interface PostLocation {
  name: string;
  latitude?: number;
  longitude?: number;
}

export interface PostEdit {
  content: string;
  editedAt: Date;
}

// ========================
// Comment Types
// ========================
export interface Comment {
  id: string;
  postId: string;
  post?: Post;
  authorId: string;
  author?: User;
  parentCommentId?: string;
  parentComment?: Comment;
  
  content: string;
  media?: PostMedia[];
  
  likesCount: number;
  repliesCount: number;
  
  isEdited: boolean;
  isPinned: boolean; // Pinned by post author
  
  mentions: string[];
  
  createdAt: Date;
  updatedAt: Date;
}

// ========================
// Like Types
// ========================
export interface Like {
  id: string;
  userId: string;
  user?: User;
  targetId: string;
  targetType: "post" | "comment";
  reactionType: ReactionType;
  createdAt: Date;
}

export enum ReactionType {
  LIKE = "LIKE",
  LOVE = "LOVE",
  HAHA = "HAHA",
  WOW = "WOW",
  SAD = "SAD",
  ANGRY = "ANGRY"
}

// ========================
// Follow Types
// ========================
export interface Follow {
  id: string;
  followerId: string;
  follower?: User;
  followingId: string;
  following?: User;
  status: FollowStatus;
  createdAt: Date;
}

export enum FollowStatus {
  PENDING = "PENDING",
  ACCEPTED = "ACCEPTED",
  BLOCKED = "BLOCKED"
}

// ========================
// Notification Types
// ========================
export interface Notification {
  id: string;
  recipientId: string;
  recipient?: User;
  senderId: string;
  sender?: User;
  type: NotificationType;
  entityId?: string;
  entityType?: string;
  message: string;
  isRead: boolean;
  readAt?: Date;
  createdAt: Date;
}

export enum NotificationType {
  LIKE_POST = "LIKE_POST",
  LIKE_COMMENT = "LIKE_COMMENT",
  COMMENT_POST = "COMMENT_POST",
  REPLY_COMMENT = "REPLY_COMMENT",
  FOLLOW = "FOLLOW",
  FOLLOW_REQUEST = "FOLLOW_REQUEST",
  FOLLOW_ACCEPTED = "FOLLOW_ACCEPTED",
  MENTION_POST = "MENTION_POST",
  MENTION_COMMENT = "MENTION_COMMENT",
  REPOST = "REPOST",
  QUOTE = "QUOTE",
  MESSAGE = "MESSAGE"
}

// ========================
// Message Types
// ========================
export interface Conversation {
  id: string;
  type: ConversationType;
  name?: string; // For group conversations
  avatar?: string;
  description?: string;
  participants: ConversationParticipant[];
  lastMessage?: Message;
  unreadCount?: number;
  isArchived: boolean;
  createdAt: Date;
  updatedAt: Date;
}

export enum ConversationType {
  DIRECT = "DIRECT",
  GROUP = "GROUP"
}

export interface ConversationParticipant {
  userId: string;
  user?: User;
  conversationId: string;
  role: "admin" | "member";
  joinedAt: Date;
  leftAt?: Date;
  lastReadAt?: Date;
  isActive: boolean;
  isMuted: boolean;
}

export interface Message {
  id: string;
  conversationId: string;
  senderId: string;
  sender?: User;
  
  content?: string;
  media?: MessageMedia[];
  
  type: MessageType;
  
  replyToId?: string;
  replyTo?: Message;
  
  isEdited: boolean;
  isDeleted: boolean;
  deletedFor?: string[]; // userIds who deleted this message
  
  reactions: MessageReaction[];
  readBy: MessageReadStatus[];
  
  createdAt: Date;
  updatedAt: Date;
}

export enum MessageType {
  TEXT = "TEXT",
  MEDIA = "MEDIA",
  POST_SHARE = "POST_SHARE",
  SYSTEM = "SYSTEM" // "User joined the group", etc.
}

export interface MessageMedia {
  url: string;
  type: "image" | "video" | "audio" | "file";
  name?: string;
  size: number;
  mimeType: string;
}

export interface MessageReaction {
  userId: string;
  emoji: string;
  createdAt: Date;
}

export interface MessageReadStatus {
  userId: string;
  readAt: Date;
}

// ========================
// Hashtag Types
// ========================
export interface Hashtag {
  id: string;
  name: string;
  postsCount: number;
  trendingScore: number;
  createdAt: Date;
}

// ========================
// Search Types
// ========================
export interface SearchResult {
  users: User[];
  posts: Post[];
  hashtags: Hashtag[];
}
```

---

## User Service

### src/services/UserService.ts

```typescript
import { prisma } from "../config/database";
import { redis } from "../config/redis";
import bcrypt from "bcryptjs";
import type { User } from "../types/models";

export class UserService {
  async getUserById(userId: string): Promise<User | null> {
    // Try cache first
    const cached = await redis.get(`user:${userId}`);
    if (cached) return JSON.parse(cached);
    
    const user = await prisma.user.findUnique({
      where: { id: userId },
      select: {
        id: true,
        username: true,
        email: true,
        displayName: true,
        bio: true,
        avatar: true,
        coverImage: true,
        website: true,
        location: true,
        isVerified: true,
        isPrivate: true,
        role: true,
        followersCount: true,
        followingCount: true,
        postsCount: true,
        createdAt: true,
        updatedAt: true
      }
    });
    
    if (user) {
      await redis.setex(`user:${userId}`, 300, JSON.stringify(user)); // cache 5 min
    }
    
    return user as User | null;
  }
  
  async getUserByUsername(username: string): Promise<User | null> {
    return prisma.user.findUnique({
      where: { username }
    }) as unknown as User | null;
  }
  
  async updateProfile(
    userId: string,
    data: {
      displayName?: string;
      bio?: string;
      website?: string;
      location?: string;
      dateOfBirth?: Date;
    }
  ): Promise<User> {
    const user = await prisma.user.update({
      where: { id: userId },
      data: { ...data, updatedAt: new Date() }
    });
    
    // Invalidate cache
    await redis.del(`user:${userId}`);
    
    return user as unknown as User;
  }
  
  async updateAvatar(userId: string, avatarUrl: string): Promise<void> {
    await prisma.user.update({
      where: { id: userId },
      data: { avatar: avatarUrl }
    });
    
    await redis.del(`user:${userId}`);
  }
  
  async updateCoverImage(userId: string, coverUrl: string): Promise<void> {
    await prisma.user.update({
      where: { id: userId },
      data: { coverImage: coverUrl }
    });
    
    await redis.del(`user:${userId}`);
  }
  
  async changePassword(
    userId: string,
    currentPassword: string,
    newPassword: string
  ): Promise<void> {
    const user = await prisma.user.findUnique({ where: { id: userId } });
    if (!user) throw new Error("User not found");
    
    const isValid = await bcrypt.compare(currentPassword, user.passwordHash);
    if (!isValid) throw new Error("รหัสผ่านปัจจุบันไม่ถูกต้อง");
    
    const passwordHash = await bcrypt.hash(newPassword, 12);
    await prisma.user.update({
      where: { id: userId },
      data: { passwordHash }
    });
  }
  
  async deactivateAccount(userId: string): Promise<void> {
    await prisma.user.update({
      where: { id: userId },
      data: { isActive: false }
    });
    
    await redis.del(`user:${userId}`);
  }
  
  async searchUsers(query: string, limit: number = 10): Promise<User[]> {
    const users = await prisma.user.findMany({
      where: {
        isActive: true,
        OR: [
          { username: { contains: query, mode: "insensitive" } },
          { displayName: { contains: query, mode: "insensitive" } }
        ]
      },
      take: limit,
      orderBy: [
        { followersCount: "desc" },
        { isVerified: "desc" }
      ]
    });
    
    return users as unknown as User[];
  }
  
  async getSuggestedUsers(userId: string, limit: number = 10): Promise<User[]> {
    // Get users followed by people I follow (FOF - Friends of Friends)
    const following = await prisma.follow.findMany({
      where: { followerId: userId, status: "ACCEPTED" },
      select: { followingId: true }
    });
    
    const followingIds = following.map(f => f.followingId);
    
    if (followingIds.length === 0) {
      // Fallback: suggest popular users
      return prisma.user.findMany({
        where: {
          id: { not: userId },
          isActive: true
        },
        orderBy: { followersCount: "desc" },
        take: limit
      }) as unknown as User[];
    }
    
    // Friends of friends
    const fofFollowing = await prisma.follow.findMany({
      where: {
        followerId: { in: followingIds },
        followingId: { notIn: [...followingIds, userId] },
        status: "ACCEPTED"
      },
      distinct: ["followingId"],
      take: limit * 2,
      select: { followingId: true, followerId: true }
    });
    
    const suggestedIds = [...new Set(fofFollowing.map(f => f.followingId))];
    
    const users = await prisma.user.findMany({
      where: {
        id: { in: suggestedIds.slice(0, limit) },
        isActive: true
      }
    });
    
    return users as unknown as User[];
  }
  
  async getFollowers(userId: string, page: number = 1, limit: number = 20) {
    const [total, followers] = await Promise.all([
      prisma.follow.count({ where: { followingId: userId, status: "ACCEPTED" } }),
      prisma.follow.findMany({
        where: { followingId: userId, status: "ACCEPTED" },
        skip: (page - 1) * limit,
        take: limit,
        include: {
          follower: {
            select: {
              id: true,
              username: true,
              displayName: true,
              avatar: true,
              isVerified: true,
              bio: true,
              followersCount: true
            }
          }
        },
        orderBy: { createdAt: "desc" }
      })
    ]);
    
    return {
      followers: followers.map(f => f.follower),
      total,
      page,
      totalPages: Math.ceil(total / limit)
    };
  }
  
  async getFollowing(userId: string, page: number = 1, limit: number = 20) {
    const [total, following] = await Promise.all([
      prisma.follow.count({ where: { followerId: userId, status: "ACCEPTED" } }),
      prisma.follow.findMany({
        where: { followerId: userId, status: "ACCEPTED" },
        skip: (page - 1) * limit,
        take: limit,
        include: {
          following: {
            select: {
              id: true,
              username: true,
              displayName: true,
              avatar: true,
              isVerified: true,
              bio: true,
              followersCount: true
            }
          }
        },
        orderBy: { createdAt: "desc" }
      })
    ]);
    
    return {
      following: following.map(f => f.following),
      total,
      page,
      totalPages: Math.ceil(total / limit)
    };
  }
  
  async updateNotificationSettings(
    userId: string,
    settings: Partial<User["notificationSettings"]>
  ): Promise<void> {
    await prisma.user.update({
      where: { id: userId },
      data: {
        notificationSettings: settings
      }
    });
  }
  
  async blockUser(userId: string, targetId: string): Promise<void> {
    // Create block record
    await prisma.userBlock.upsert({
      where: {
        blockerId_blockedId: {
          blockerId: userId,
          blockedId: targetId
        }
      },
      create: { blockerId: userId, blockedId: targetId },
      update: {}
    });
    
    // Remove follow relationships
    await prisma.follow.deleteMany({
      where: {
        OR: [
          { followerId: userId, followingId: targetId },
          { followerId: targetId, followingId: userId }
        ]
      }
    });
  }
  
  async unblockUser(userId: string, targetId: string): Promise<void> {
    await prisma.userBlock.delete({
      where: {
        blockerId_blockedId: {
          blockerId: userId,
          blockedId: targetId
        }
      }
    });
  }
  
  async getBlockedUsers(userId: string): Promise<User[]> {
    const blocks = await prisma.userBlock.findMany({
      where: { blockerId: userId },
      include: {
        blocked: {
          select: {
            id: true,
            username: true,
            displayName: true,
            avatar: true
          }
        }
      }
    });
    
    return blocks.map(b => b.blocked) as unknown as User[];
  }
}

export const userService = new UserService();
```

---

## Post Service

### src/services/PostService.ts

```typescript
import { prisma } from "../config/database";
import { redis } from "../config/redis";
import { notificationService } from "./NotificationService";
import type { Post, PostType, PostVisibility } from "../types/models";

interface CreatePostData {
  content?: string;
  mediaUrls?: string[];
  visibility?: PostVisibility;
  type?: PostType;
  replyToId?: string;
  originalPostId?: string;
  location?: { name: string; latitude?: number; longitude?: number };
  isSensitive?: boolean;
}

export class PostService {
  async createPost(authorId: string, data: CreatePostData): Promise<Post> {
    if (!data.content && (!data.mediaUrls || data.mediaUrls.length === 0)) {
      throw new Error("โพสต์ต้องมีเนื้อหาหรือรูปภาพ");
    }
    
    // Extract hashtags and mentions
    const hashtags = this.extractHashtags(data.content || "");
    const mentions = this.extractMentions(data.content || "");
    
    // Validate reply/repost
    if (data.replyToId) {
      const originalPost = await prisma.post.findUnique({
        where: { id: data.replyToId }
      });
      if (!originalPost) throw new Error("ไม่พบโพสต์ที่ต้องการตอบกลับ");
    }
    
    const post = await prisma.$transaction(async (tx) => {
      const newPost = await tx.post.create({
        data: {
          authorId,
          content: data.content,
          type: data.type || "POST",
          visibility: data.visibility || "PUBLIC",
          replyToId: data.replyToId,
          originalPostId: data.originalPostId,
          hashtags,
          mentions,
          location: data.location ? JSON.stringify(data.location) : undefined,
          isSensitive: data.isSensitive || false,
          media: data.mediaUrls
            ? {
                create: data.mediaUrls.map((url, index) => ({
                  url,
                  type: this.detectMediaType(url),
                  sortOrder: index,
                  size: 0,
                  mimeType: "image/jpeg"
                }))
              }
            : undefined
        },
        include: {
          author: {
            select: {
              id: true,
              username: true,
              displayName: true,
              avatar: true,
              isVerified: true
            }
          },
          media: true,
          replyTo: {
            include: {
              author: {
                select: { id: true, username: true, displayName: true }
              }
            }
          }
        }
      });
      
      // Update user post count
      await tx.user.update({
        where: { id: authorId },
        data: { postsCount: { increment: 1 } }
      });
      
      // If reply, update reply count
      if (data.replyToId) {
        await tx.post.update({
          where: { id: data.replyToId },
          data: { commentsCount: { increment: 1 } }
        });
      }
      
      // If repost, update repost count
      if (data.originalPostId && (data.type === "REPOST" || data.type === "QUOTE")) {
        await tx.post.update({
          where: { id: data.originalPostId },
          data: { repostsCount: { increment: 1 } }
        });
      }
      
      // Update hashtag counts
      for (const tag of hashtags) {
        await tx.hashtag.upsert({
          where: { name: tag },
          create: { name: tag, postsCount: 1 },
          update: { postsCount: { increment: 1 } }
        });
      }
      
      return newPost;
    });
    
    // Send notifications to mentioned users (async)
    this.notifyMentions(authorId, mentions, post.id).catch(console.error);
    
    // If reply, notify original post author
    if (data.replyToId) {
      const originalPost = await prisma.post.findUnique({
        where: { id: data.replyToId }
      });
      if (originalPost && originalPost.authorId !== authorId) {
        notificationService.create({
          recipientId: originalPost.authorId,
          senderId: authorId,
          type: "COMMENT_POST",
          entityId: post.id,
          entityType: "post"
        }).catch(console.error);
      }
    }
    
    // Invalidate feeds (async)
    this.invalidateFeeds(authorId).catch(console.error);
    
    return post as unknown as Post;
  }
  
  async getPostById(postId: string, viewerId?: string): Promise<Post> {
    const post = await prisma.post.findUnique({
      where: { id: postId },
      include: {
        author: {
          select: {
            id: true,
            username: true,
            displayName: true,
            avatar: true,
            isVerified: true
          }
        },
        media: { orderBy: { sortOrder: "asc" } },
        replyTo: {
          include: {
            author: { select: { id: true, username: true, displayName: true } }
          }
        },
        originalPost: {
          include: {
            author: {
              select: {
                id: true,
                username: true,
                displayName: true,
                avatar: true,
                isVerified: true
              }
            },
            media: true
          }
        }
      }
    });
    
    if (!post) throw new Error("ไม่พบโพสต์");
    
    // Increment view count (debounced via Redis)
    if (viewerId) {
      const viewKey = `post:view:${postId}:${viewerId}`;
      const hasViewed = await redis.get(viewKey);
      
      if (!hasViewed) {
        await redis.setex(viewKey, 3600, "1"); // 1 hour debounce
        await prisma.post.update({
          where: { id: postId },
          data: { viewsCount: { increment: 1 } }
        });
      }
    }
    
    return post as unknown as Post;
  }
  
  async getUserPosts(
    userId: string,
    viewerId?: string,
    page: number = 1,
    limit: number = 20,
    type?: PostType
  ) {
    // Check if viewer can see private posts
    const isOwner = viewerId === userId;
    let canSeePrivate = isOwner;
    
    if (!isOwner && viewerId) {
      const follow = await prisma.follow.findFirst({
        where: {
          followerId: viewerId,
          followingId: userId,
          status: "ACCEPTED"
        }
      });
      canSeePrivate = !!follow;
    }
    
    const where: any = {
      authorId: userId,
      NOT: { type: "REPLY" } // Default: don't show replies in main feed
    };
    
    if (type) where.type = type;
    
    if (!canSeePrivate) {
      where.visibility = "PUBLIC";
    } else if (!isOwner) {
      where.visibility = { in: ["PUBLIC", "FOLLOWERS_ONLY"] };
    }
    
    const [total, posts] = await Promise.all([
      prisma.post.count({ where }),
      prisma.post.findMany({
        where,
        orderBy: { createdAt: "desc" },
        skip: (page - 1) * limit,
        take: limit,
        include: {
          author: {
            select: {
              id: true,
              username: true,
              displayName: true,
              avatar: true,
              isVerified: true
            }
          },
          media: { orderBy: { sortOrder: "asc" } },
          _count: { select: { likes: true, comments: true } }
        }
      })
    ]);
    
    return {
      posts,
      total,
      page,
      totalPages: Math.ceil(total / limit)
    };
  }
  
  async updatePost(
    postId: string,
    authorId: string,
    data: { content?: string; isSensitive?: boolean }
  ): Promise<Post> {
    const post = await prisma.post.findUnique({ where: { id: postId } });
    
    if (!post) throw new Error("ไม่พบโพสต์");
    if (post.authorId !== authorId) throw new Error("ไม่มีสิทธิ์แก้ไขโพสต์นี้");
    
    // Store edit history
    const editHistory = [
      ...(post.editHistory as any[] || []),
      { content: post.content, editedAt: new Date().toISOString() }
    ];
    
    const updated = await prisma.post.update({
      where: { id: postId },
      data: {
        ...data,
        isEdited: true,
        editHistory: JSON.stringify(editHistory),
        hashtags: data.content ? this.extractHashtags(data.content) : undefined,
        mentions: data.content ? this.extractMentions(data.content) : undefined
      },
      include: {
        author: {
          select: { id: true, username: true, displayName: true, avatar: true }
        },
        media: true
      }
    });
    
    return updated as unknown as Post;
  }
  
  async deletePost(postId: string, userId: string): Promise<void> {
    const post = await prisma.post.findUnique({
      where: { id: postId },
      include: { author: true }
    });
    
    if (!post) throw new Error("ไม่พบโพสต์");
    
    const isAuthor = post.authorId === userId;
    const isModAdmin = await prisma.user.findFirst({
      where: { id: userId, role: { in: ["MODERATOR", "ADMIN"] } }
    });
    
    if (!isAuthor && !isModAdmin) {
      throw new Error("ไม่มีสิทธิ์ลบโพสต์นี้");
    }
    
    await prisma.$transaction(async (tx) => {
      await tx.post.delete({ where: { id: postId } });
      
      await tx.user.update({
        where: { id: post.authorId },
        data: { postsCount: { decrement: 1 } }
      });
      
      // Update reply count on parent
      if (post.replyToId) {
        await tx.post.update({
          where: { id: post.replyToId },
          data: { commentsCount: { decrement: 1 } }
        });
      }
    });
  }
  
  async pinPost(postId: string, userId: string): Promise<void> {
    // Unpin current pinned post
    await prisma.post.updateMany({
      where: { authorId: userId, isPinned: true },
      data: { isPinned: false }
    });
    
    // Pin new post
    await prisma.post.update({
      where: { id: postId, authorId: userId },
      data: { isPinned: true }
    });
  }
  
  async getPostReplies(
    postId: string,
    page: number = 1,
    limit: number = 20
  ) {
    const [total, replies] = await Promise.all([
      prisma.post.count({ where: { replyToId: postId } }),
      prisma.post.findMany({
        where: { replyToId: postId },
        orderBy: [
          { isPinned: "desc" },
          { likesCount: "desc" },
          { createdAt: "asc" }
        ],
        skip: (page - 1) * limit,
        take: limit,
        include: {
          author: {
            select: {
              id: true,
              username: true,
              displayName: true,
              avatar: true,
              isVerified: true
            }
          },
          media: true,
          _count: { select: { likes: true, comments: true } }
        }
      })
    ]);
    
    return { replies, total, page };
  }
  
  async repost(postId: string, userId: string): Promise<Post> {
    // Check if already reposted
    const existing = await prisma.post.findFirst({
      where: {
        authorId: userId,
        originalPostId: postId,
        type: "REPOST"
      }
    });
    
    if (existing) {
      // Undo repost
      await prisma.post.delete({ where: { id: existing.id } });
      await prisma.post.update({
        where: { id: postId },
        data: { repostsCount: { decrement: 1 } }
      });
      throw new Error("UNDO_REPOST");
    }
    
    return this.createPost(userId, {
      type: "REPOST",
      originalPostId: postId
    });
  }
  
  private extractHashtags(content: string): string[] {
    const matches = content.match(/#[\w฀-๿]+/g) || [];
    return [...new Set(matches.map(tag => tag.slice(1).toLowerCase()))];
  }
  
  private extractMentions(content: string): string[] {
    const matches = content.match(/@[\w]+/g) || [];
    return [...new Set(matches.map(m => m.slice(1).toLowerCase()))];
  }
  
  private detectMediaType(url: string): "image" | "video" | "gif" {
    if (url.endsWith(".gif")) return "gif";
    if (/\.(mp4|mov|avi|webm)$/.test(url)) return "video";
    return "image";
  }
  
  private async notifyMentions(
    senderId: string,
    mentions: string[],
    postId: string
  ): Promise<void> {
    for (const username of mentions) {
      const user = await prisma.user.findUnique({ where: { username } });
      if (user && user.id !== senderId) {
        await notificationService.create({
          recipientId: user.id,
          senderId,
          type: "MENTION_POST",
          entityId: postId,
          entityType: "post"
        });
      }
    }
  }
  
  private async invalidateFeeds(userId: string): Promise<void> {
    // Invalidate feeds of followers
    const followers = await prisma.follow.findMany({
      where: { followingId: userId, status: "ACCEPTED" },
      select: { followerId: true }
    });
    
    const pipeline = redis.pipeline();
    
    for (const { followerId } of followers) {
      pipeline.del(`feed:${followerId}`);
    }
    
    await pipeline.exec();
  }
}

export const postService = new PostService();
```

---

## Follow Service

### src/services/FollowService.ts

```typescript
import { prisma } from "../config/database";
import { notificationService } from "./NotificationService";
import type { Follow, FollowStatus } from "../types/models";

export class FollowService {
  async follow(followerId: string, followingId: string): Promise<Follow> {
    if (followerId === followingId) {
      throw new Error("ไม่สามารถติดตามตัวเองได้");
    }
    
    // Check if blocked
    const isBlocked = await prisma.userBlock.findFirst({
      where: {
        OR: [
          { blockerId: followerId, blockedId: followingId },
          { blockerId: followingId, blockedId: followerId }
        ]
      }
    });
    
    if (isBlocked) throw new Error("ไม่สามารถติดตามผู้ใช้นี้ได้");
    
    // Check existing follow
    const existing = await prisma.follow.findFirst({
      where: { followerId, followingId }
    });
    
    if (existing) {
      if (existing.status === "ACCEPTED") {
        throw new Error("คุณกำลังติดตามผู้ใช้นี้อยู่แล้ว");
      }
      return existing as unknown as Follow;
    }
    
    // Check if target user is private
    const targetUser = await prisma.user.findUnique({
      where: { id: followingId }
    });
    
    if (!targetUser) throw new Error("ไม่พบผู้ใช้");
    
    const status: FollowStatus = targetUser.isPrivate ? "PENDING" : "ACCEPTED";
    
    const follow = await prisma.$transaction(async (tx) => {
      const newFollow = await tx.follow.create({
        data: { followerId, followingId, status }
      });
      
      if (status === "ACCEPTED") {
        await tx.user.update({
          where: { id: followerId },
          data: { followingCount: { increment: 1 } }
        });
        await tx.user.update({
          where: { id: followingId },
          data: { followersCount: { increment: 1 } }
        });
      }
      
      return newFollow;
    });
    
    // Send notification
    await notificationService.create({
      recipientId: followingId,
      senderId: followerId,
      type: status === "PENDING" ? "FOLLOW_REQUEST" : "FOLLOW",
      entityId: follow.id,
      entityType: "follow"
    });
    
    return follow as unknown as Follow;
  }
  
  async unfollow(followerId: string, followingId: string): Promise<void> {
    const follow = await prisma.follow.findFirst({
      where: { followerId, followingId }
    });
    
    if (!follow) throw new Error("คุณไม่ได้ติดตามผู้ใช้นี้");
    
    await prisma.$transaction(async (tx) => {
      await tx.follow.delete({ where: { id: follow.id } });
      
      if (follow.status === "ACCEPTED") {
        await tx.user.update({
          where: { id: followerId },
          data: { followingCount: { decrement: 1 } }
        });
        await tx.user.update({
          where: { id: followingId },
          data: { followersCount: { decrement: 1 } }
        });
      }
    });
  }
  
  async acceptFollowRequest(userId: string, followerId: string): Promise<void> {
    const follow = await prisma.follow.findFirst({
      where: { followerId, followingId: userId, status: "PENDING" }
    });
    
    if (!follow) throw new Error("ไม่พบคำขอติดตาม");
    
    await prisma.$transaction(async (tx) => {
      await tx.follow.update({
        where: { id: follow.id },
        data: { status: "ACCEPTED" }
      });
      
      await tx.user.update({
        where: { id: followerId },
        data: { followingCount: { increment: 1 } }
      });
      await tx.user.update({
        where: { id: userId },
        data: { followersCount: { increment: 1 } }
      });
    });
    
    await notificationService.create({
      recipientId: followerId,
      senderId: userId,
      type: "FOLLOW_ACCEPTED",
      entityType: "follow"
    });
  }
  
  async rejectFollowRequest(userId: string, followerId: string): Promise<void> {
    await prisma.follow.deleteMany({
      where: { followerId, followingId: userId, status: "PENDING" }
    });
  }
  
  async removeFollower(userId: string, followerId: string): Promise<void> {
    await this.unfollow(followerId, userId);
  }
  
  async getFollowStatus(
    viewerId: string,
    targetId: string
  ): Promise<"following" | "pending" | "not_following" | "blocked"> {
    const block = await prisma.userBlock.findFirst({
      where: {
        OR: [
          { blockerId: viewerId, blockedId: targetId },
          { blockerId: targetId, blockedId: viewerId }
        ]
      }
    });
    
    if (block) return "blocked";
    
    const follow = await prisma.follow.findFirst({
      where: { followerId: viewerId, followingId: targetId }
    });
    
    if (!follow) return "not_following";
    if (follow.status === "PENDING") return "pending";
    return "following";
  }
  
  async getMutualFollowers(userId1: string, userId2: string): Promise<string[]> {
    const [followers1, followers2] = await Promise.all([
      prisma.follow.findMany({
        where: { followingId: userId1, status: "ACCEPTED" },
        select: { followerId: true }
      }),
      prisma.follow.findMany({
        where: { followingId: userId2, status: "ACCEPTED" },
        select: { followerId: true }
      })
    ]);
    
    const set1 = new Set(followers1.map(f => f.followerId));
    return followers2
      .filter(f => set1.has(f.followerId))
      .map(f => f.followerId);
  }
}

export const followService = new FollowService();
```

---

## Feed Service

### src/services/FeedService.ts

```typescript
import { prisma } from "../config/database";
import { redis } from "../config/redis";
import type { Post } from "../types/models";

export class FeedService {
  private readonly FEED_CACHE_TTL = 300; // 5 minutes
  private readonly FEED_MAX_POSTS = 500;
  
  async getHomeFeed(
    userId: string,
    page: number = 1,
    limit: number = 20
  ): Promise<{ posts: Post[]; hasMore: boolean }> {
    // Try to get from cache
    const cachedFeed = await this.getCachedFeed(userId);
    
    if (cachedFeed.length > 0) {
      const start = (page - 1) * limit;
      const end = start + limit;
      const posts = cachedFeed.slice(start, end);
      
      if (posts.length > 0) {
        const fullPosts = await this.hydratePosts(posts);
        return {
          posts: fullPosts,
          hasMore: end < cachedFeed.length
        };
      }
    }
    
    // Build feed from database
    return this.buildFeedFromDb(userId, page, limit);
  }
  
  async buildFeedFromDb(
    userId: string,
    page: number = 1,
    limit: number = 20
  ): Promise<{ posts: Post[]; hasMore: boolean }> {
    // Get users that this person follows
    const following = await prisma.follow.findMany({
      where: { followerId: userId, status: "ACCEPTED" },
      select: { followingId: true }
    });
    
    const followingIds = following.map(f => f.followingId);
    includeIds = [...followingIds, userId]; // Include own posts
    
    // Get blocked users
    const blocked = await prisma.userBlock.findMany({
      where: {
        OR: [
          { blockerId: userId },
          { blockedId: userId }
        ]
      },
      select: { blockerId: true, blockedId: true }
    });
    
    const blockedIds = blocked.flatMap(b => [b.blockerId, b.blockedId])
      .filter(id => id !== userId);
    
    const [total, posts] = await Promise.all([
      prisma.post.count({
        where: {
          authorId: { in: includeIds, notIn: blockedIds },
          type: { not: "REPLY" },
          OR: [
            { visibility: "PUBLIC" },
            {
              visibility: "FOLLOWERS_ONLY",
              authorId: { in: followingIds }
            }
          ]
        }
      }),
      prisma.post.findMany({
        where: {
          authorId: { in: includeIds, notIn: blockedIds },
          type: { not: "REPLY" },
          OR: [
            { visibility: "PUBLIC" },
            {
              visibility: "FOLLOWERS_ONLY",
              authorId: { in: followingIds }
            }
          ]
        },
        orderBy: [
          { createdAt: "desc" }
        ],
        skip: (page - 1) * limit,
        take: limit,
        include: {
          author: {
            select: {
              id: true,
              username: true,
              displayName: true,
              avatar: true,
              isVerified: true
            }
          },
          media: { orderBy: { sortOrder: "asc" } },
          originalPost: {
            include: {
              author: {
                select: { id: true, username: true, displayName: true, avatar: true }
              },
              media: { orderBy: { sortOrder: "asc" } }
            }
          },
          _count: { select: { likes: true } }
        }
      })
    ]);
    
    // Cache the feed
    if (page === 1) {
      await this.cacheFeed(userId, posts.map(p => p.id));
    }
    
    return {
      posts: posts as unknown as Post[],
      hasMore: (page * limit) < total
    };
  }
  
  async getExploreFeed(
    userId: string,
    page: number = 1,
    limit: number = 20
  ): Promise<{ posts: Post[]; hasMore: boolean }> {
    // Get blocked users
    const blocked = await prisma.userBlock.findMany({
      where: { OR: [{ blockerId: userId }, { blockedId: userId }] },
      select: { blockerId: true, blockedId: true }
    });
    
    const blockedIds = blocked.flatMap(b => [b.blockerId, b.blockedId])
      .filter(id => id !== userId);
    
    // Get trending/popular posts from last 48 hours
    const since = new Date(Date.now() - 48 * 60 * 60 * 1000);
    
    const posts = await prisma.post.findMany({
      where: {
        authorId: { notIn: blockedIds },
        visibility: "PUBLIC",
        type: { not: "REPLY" },
        isSensitive: false,
        createdAt: { gte: since }
      },
      orderBy: [
        { likesCount: "desc" },
        { viewsCount: "desc" },
        { commentsCount: "desc" }
      ],
      skip: (page - 1) * limit,
      take: limit,
      include: {
        author: {
          select: {
            id: true,
            username: true,
            displayName: true,
            avatar: true,
            isVerified: true
          }
        },
        media: { orderBy: { sortOrder: "asc" }, take: 1 },
        _count: { select: { likes: true } }
      }
    });
    
    return {
      posts: posts as unknown as Post[],
      hasMore: posts.length === limit
    };
  }
  
  async getHashtagFeed(
    hashtag: string,
    page: number = 1,
    limit: number = 20
  ) {
    const [total, posts] = await Promise.all([
      prisma.post.count({
        where: {
          hashtags: { has: hashtag.toLowerCase() },
          visibility: "PUBLIC"
        }
      }),
      prisma.post.findMany({
        where: {
          hashtags: { has: hashtag.toLowerCase() },
          visibility: "PUBLIC"
        },
        orderBy: { createdAt: "desc" },
        skip: (page - 1) * limit,
        take: limit,
        include: {
          author: {
            select: {
              id: true,
              username: true,
              displayName: true,
              avatar: true,
              isVerified: true
            }
          },
          media: { take: 1 }
        }
      })
    ]);
    
    // Update trending score
    await prisma.hashtag.update({
      where: { name: hashtag.toLowerCase() },
      data: { trendingScore: { increment: 1 } }
    }).catch(() => {});
    
    return { posts, total, page };
  }
  
  async getTrendingHashtags(limit: number = 10): Promise<string[]> {
    const cached = await redis.get("trending:hashtags");
    if (cached) return JSON.parse(cached);
    
    const trends = await prisma.hashtag.findMany({
      orderBy: [{ trendingScore: "desc" }, { postsCount: "desc" }],
      take: limit,
      select: { name: true }
    });
    
    const hashtags = trends.map(t => t.name);
    await redis.setex("trending:hashtags", 3600, JSON.stringify(hashtags));
    
    return hashtags;
  }
  
  private async getCachedFeed(userId: string): Promise<string[]> {
    const cached = await redis.lrange(`feed:${userId}`, 0, -1);
    return cached;
  }
  
  private async cacheFeed(userId: string, postIds: string[]): Promise<void> {
    if (postIds.length === 0) return;
    
    const pipeline = redis.pipeline();
    pipeline.del(`feed:${userId}`);
    
    const ids = postIds.slice(0, this.FEED_MAX_POSTS);
    if (ids.length > 0) {
      pipeline.rpush(`feed:${userId}`, ...ids);
      pipeline.expire(`feed:${userId}`, this.FEED_CACHE_TTL);
    }
    
    await pipeline.exec();
  }
  
  private async hydratePosts(postIds: string[]): Promise<Post[]> {
    const posts = await prisma.post.findMany({
      where: { id: { in: postIds } },
      include: {
        author: {
          select: {
            id: true,
            username: true,
            displayName: true,
            avatar: true,
            isVerified: true
          }
        },
        media: { orderBy: { sortOrder: "asc" } }
      }
    });
    
    // Preserve original order
    const postMap = new Map(posts.map(p => [p.id, p]));
    return postIds
      .map(id => postMap.get(id))
      .filter(Boolean) as unknown as Post[];
  }
}

let includeIds: string[] = [];
export const feedService = new FeedService();
```

---

## Notification Service

### src/services/NotificationService.ts

```typescript
import { prisma } from "../config/database";
import { redis } from "../config/redis";
import type { Notification, NotificationType } from "../types/models";

interface CreateNotificationData {
  recipientId: string;
  senderId: string;
  type: NotificationType;
  entityId?: string;
  entityType?: string;
}

export class NotificationService {
  async create(data: CreateNotificationData): Promise<Notification> {
    // Don't notify yourself
    if (data.recipientId === data.senderId) {
      return {} as Notification;
    }
    
    // Check notification settings
    const recipient = await prisma.user.findUnique({
      where: { id: data.recipientId },
      select: { notificationSettings: true }
    });
    
    if (!recipient) return {} as Notification;
    
    const settings = recipient.notificationSettings as any;
    
    // Check if this notification type is enabled
    const isEnabled = this.isNotificationEnabled(data.type, settings);
    if (!isEnabled) return {} as Notification;
    
    const message = await this.buildMessage(data);
    
    const notification = await prisma.notification.create({
      data: {
        ...data,
        message,
        isRead: false
      },
      include: {
        sender: {
          select: {
            id: true,
            username: true,
            displayName: true,
            avatar: true
          }
        }
      }
    });
    
    // Invalidate unread count cache
    await redis.del(`notifications:unread:${data.recipientId}`);
    
    // Push real-time notification via WebSocket
    await this.pushRealTimeNotification(data.recipientId, notification);
    
    return notification as unknown as Notification;
  }
  
  async getNotifications(
    userId: string,
    page: number = 1,
    limit: number = 20
  ) {
    const [total, notifications] = await Promise.all([
      prisma.notification.count({ where: { recipientId: userId } }),
      prisma.notification.findMany({
        where: { recipientId: userId },
        orderBy: { createdAt: "desc" },
        skip: (page - 1) * limit,
        take: limit,
        include: {
          sender: {
            select: {
              id: true,
              username: true,
              displayName: true,
              avatar: true,
              isVerified: true
            }
          }
        }
      })
    ]);
    
    return { notifications, total, page };
  }
  
  async getUnreadCount(userId: string): Promise<number> {
    const cached = await redis.get(`notifications:unread:${userId}`);
    if (cached) return parseInt(cached);
    
    const count = await prisma.notification.count({
      where: { recipientId: userId, isRead: false }
    });
    
    await redis.setex(`notifications:unread:${userId}`, 60, count.toString());
    
    return count;
  }
  
  async markAsRead(userId: string, notificationId?: string): Promise<void> {
    const where: any = { recipientId: userId };
    if (notificationId) where.id = notificationId;
    
    await prisma.notification.updateMany({
      where,
      data: { isRead: true, readAt: new Date() }
    });
    
    await redis.del(`notifications:unread:${userId}`);
  }
  
  async markAllAsRead(userId: string): Promise<void> {
    await this.markAsRead(userId);
  }
  
  async deleteNotification(userId: string, notificationId: string): Promise<void> {
    await prisma.notification.deleteMany({
      where: { id: notificationId, recipientId: userId }
    });
  }
  
  private isNotificationEnabled(
    type: NotificationType,
    settings: any
  ): boolean {
    if (!settings) return true;
    
    const typeToSetting: Record<string, string> = {
      LIKE_POST: "likes",
      LIKE_COMMENT: "likes",
      COMMENT_POST: "comments",
      REPLY_COMMENT: "comments",
      FOLLOW: "follows",
      FOLLOW_REQUEST: "follows",
      FOLLOW_ACCEPTED: "follows",
      MENTION_POST: "mentions",
      MENTION_COMMENT: "mentions",
      MESSAGE: "messages"
    };
    
    const settingKey = typeToSetting[type];
    return settingKey ? settings[settingKey] !== false : true;
  }
  
  private async buildMessage(data: CreateNotificationData): Promise<string> {
    const sender = await prisma.user.findUnique({
      where: { id: data.senderId },
      select: { displayName: true, username: true }
    });
    
    const name = sender?.displayName || sender?.username || "Someone";
    
    const messages: Record<NotificationType, string> = {
      LIKE_POST: `${name} ถูกใจโพสต์ของคุณ`,
      LIKE_COMMENT: `${name} ถูกใจความคิดเห็นของคุณ`,
      COMMENT_POST: `${name} แสดงความคิดเห็นในโพสต์ของคุณ`,
      REPLY_COMMENT: `${name} ตอบกลับความคิดเห็นของคุณ`,
      FOLLOW: `${name} เริ่มติดตามคุณ`,
      FOLLOW_REQUEST: `${name} ขอติดตามคุณ`,
      FOLLOW_ACCEPTED: `${name} ยอมรับคำขอติดตามของคุณ`,
      MENTION_POST: `${name} กล่าวถึงคุณในโพสต์`,
      MENTION_COMMENT: `${name} กล่าวถึงคุณในความคิดเห็น`,
      REPOST: `${name} รีโพสต์โพสต์ของคุณ`,
      QUOTE: `${name} อ้างถึงโพสต์ของคุณ`,
      MESSAGE: `${name} ส่งข้อความถึงคุณ`
    };
    
    return messages[data.type] || `${name} interacted with your content`;
  }
  
  private async pushRealTimeNotification(
    userId: string,
    notification: any
  ): Promise<void> {
    // Publish to Redis pub/sub for WebSocket delivery
    await redis.publish(
      `user:${userId}:notifications`,
      JSON.stringify(notification)
    );
  }
}

export const notificationService = new NotificationService();
```

---

## Message Service

### src/services/MessageService.ts

```typescript
import { prisma } from "../config/database";
import { redis } from "../config/redis";
import type { Message, Conversation, MessageType } from "../types/models";

export class MessageService {
  async getOrCreateDirectConversation(
    userId1: string,
    userId2: string
  ): Promise<Conversation> {
    // Find existing direct conversation
    const existing = await prisma.conversation.findFirst({
      where: {
        type: "DIRECT",
        participants: {
          every: { userId: { in: [userId1, userId2] }, isActive: true }
        }
      },
      include: {
        participants: {
          include: {
            user: {
              select: { id: true, username: true, displayName: true, avatar: true }
            }
          }
        },
        lastMessage: {
          include: {
            sender: { select: { id: true, username: true } }
          }
        }
      }
    });
    
    if (existing && existing.participants.length === 2) {
      return existing as unknown as Conversation;
    }
    
    // Create new conversation
    const conversation = await prisma.conversation.create({
      data: {
        type: "DIRECT",
        participants: {
          create: [
            { userId: userId1, role: "member" },
            { userId: userId2, role: "member" }
          ]
        }
      },
      include: {
        participants: {
          include: {
            user: {
              select: { id: true, username: true, displayName: true, avatar: true }
            }
          }
        }
      }
    });
    
    return conversation as unknown as Conversation;
  }
  
  async createGroupConversation(
    creatorId: string,
    name: string,
    participantIds: string[]
  ): Promise<Conversation> {
    if (participantIds.length < 2) {
      throw new Error("กลุ่มต้องมีสมาชิกอย่างน้อย 2 คน");
    }
    
    const allParticipants = [...new Set([creatorId, ...participantIds])];
    
    const conversation = await prisma.conversation.create({
      data: {
        type: "GROUP",
        name,
        participants: {
          create: allParticipants.map(userId => ({
            userId,
            role: userId === creatorId ? "admin" : "member"
          }))
        }
      },
      include: {
        participants: {
          include: {
            user: {
              select: { id: true, username: true, displayName: true, avatar: true }
            }
          }
        }
      }
    });
    
    // Send system message
    await this.sendSystemMessage(
      conversation.id,
      `${creatorId} สร้างกลุ่ม "${name}"`
    );
    
    return conversation as unknown as Conversation;
  }
  
  async sendMessage(
    conversationId: string,
    senderId: string,
    data: {
      content?: string;
      media?: Array<{ url: string; type: string; size: number; mimeType: string }>;
      type?: MessageType;
      replyToId?: string;
    }
  ): Promise<Message> {
    // Verify sender is participant
    const participant = await prisma.conversationParticipant.findFirst({
      where: { conversationId, userId: senderId, isActive: true }
    });
    
    if (!participant) {
      throw new Error("คุณไม่ได้เป็นสมาชิกของการสนทนานี้");
    }
    
    if (!data.content && (!data.media || data.media.length === 0)) {
      throw new Error("ข้อความต้องมีเนื้อหาหรือไฟล์แนบ");
    }
    
    const message = await prisma.$transaction(async (tx) => {
      const newMessage = await tx.message.create({
        data: {
          conversationId,
          senderId,
          content: data.content,
          type: data.type || "TEXT",
          replyToId: data.replyToId,
          media: data.media ? JSON.stringify(data.media) : undefined
        },
        include: {
          sender: {
            select: {
              id: true,
              username: true,
              displayName: true,
              avatar: true
            }
          },
          replyTo: {
            include: {
              sender: { select: { id: true, username: true } }
            }
          }
        }
      });
      
      // Update conversation's last message
      await tx.conversation.update({
        where: { id: conversationId },
        data: {
          lastMessageId: newMessage.id,
          updatedAt: new Date()
        }
      });
      
      // Update sender's last read
      await tx.conversationParticipant.update({
        where: {
          userId_conversationId: {
            userId: senderId,
            conversationId
          }
        },
        data: { lastReadAt: new Date() }
      });
      
      return newMessage;
    });
    
    // Publish to WebSocket
    await this.publishMessage(conversationId, message);
    
    return message as unknown as Message;
  }
  
  async getMessages(
    conversationId: string,
    userId: string,
    cursor?: string,
    limit: number = 50
  ) {
    // Verify access
    const participant = await prisma.conversationParticipant.findFirst({
      where: { conversationId, userId, isActive: true }
    });
    
    if (!participant) throw new Error("ไม่มีสิทธิ์เข้าถึงการสนทนานี้");
    
    const where: any = {
      conversationId,
      isDeleted: false
    };
    
    if (cursor) {
      where.createdAt = { lt: new Date(cursor) };
    }
    
    const messages = await prisma.message.findMany({
      where,
      orderBy: { createdAt: "desc" },
      take: limit,
      include: {
        sender: {
          select: {
            id: true,
            username: true,
            displayName: true,
            avatar: true
          }
        },
        replyTo: {
          include: {
            sender: { select: { id: true, username: true } }
          }
        }
      }
    });
    
    // Update last read
    await prisma.conversationParticipant.update({
      where: {
        userId_conversationId: { userId, conversationId }
      },
      data: { lastReadAt: new Date() }
    });
    
    return {
      messages: messages.reverse(),
      nextCursor: messages.length === limit
        ? messages[0].createdAt.toISOString()
        : null
    };
  }
  
  async getUserConversations(
    userId: string,
    page: number = 1,
    limit: number = 20
  ) {
    const conversations = await prisma.conversation.findMany({
      where: {
        participants: {
          some: { userId, isActive: true }
        }
      },
      orderBy: { updatedAt: "desc" },
      skip: (page - 1) * limit,
      take: limit,
      include: {
        participants: {
          where: { isActive: true },
          include: {
            user: {
              select: {
                id: true,
                username: true,
                displayName: true,
                avatar: true,
                isVerified: true
              }
            }
          }
        },
        lastMessage: {
          include: {
            sender: { select: { id: true, username: true } }
          }
        }
      }
    });
    
    // Add unread count for each conversation
    const conversationsWithUnread = await Promise.all(
      conversations.map(async (conv) => {
        const participant = conv.participants.find(p => p.userId === userId);
        const lastReadAt = participant?.lastReadAt;
        
        const unreadCount = lastReadAt
          ? await prisma.message.count({
              where: {
                conversationId: conv.id,
                createdAt: { gt: lastReadAt },
                senderId: { not: userId }
              }
            })
          : await prisma.message.count({
              where: { conversationId: conv.id, senderId: { not: userId } }
            });
        
        return { ...conv, unreadCount };
      })
    );
    
    return conversationsWithUnread;
  }
  
  async addReaction(
    messageId: string,
    userId: string,
    emoji: string
  ): Promise<void> {
    const message = await prisma.message.findUnique({ where: { id: messageId } });
    if (!message) throw new Error("ไม่พบข้อความ");
    
    const reactions = (message.reactions as any[] || []).filter(
      r => r.userId !== userId
    );
    reactions.push({ userId, emoji, createdAt: new Date() });
    
    await prisma.message.update({
      where: { id: messageId },
      data: { reactions: JSON.stringify(reactions) }
    });
  }
  
  async deleteMessage(messageId: string, userId: string): Promise<void> {
    const message = await prisma.message.findUnique({
      where: { id: messageId }
    });
    
    if (!message) throw new Error("ไม่พบข้อความ");
    if (message.senderId !== userId) throw new Error("ไม่มีสิทธิ์ลบข้อความนี้");
    
    await prisma.message.update({
      where: { id: messageId },
      data: { isDeleted: true, content: null }
    });
  }
  
  private async sendSystemMessage(
    conversationId: string,
    content: string
  ): Promise<void> {
    await prisma.message.create({
      data: {
        conversationId,
        senderId: "system",
        content,
        type: "SYSTEM"
      }
    });
  }
  
  private async publishMessage(
    conversationId: string,
    message: any
  ): Promise<void> {
    await redis.publish(
      `conversation:${conversationId}`,
      JSON.stringify(message)
    );
  }
}

export const messageService = new MessageService();
```

---

## WebSocket Handler

### src/websocket/index.ts

```typescript
import { Server as SocketServer } from "socket.io";
import { Server as HttpServer } from "http";
import { redis } from "../config/redis";
import { authService } from "../services/AuthService";
import type { Socket } from "socket.io";

interface AuthenticatedSocket extends Socket {
  userId: string;
}

export function setupWebSocket(httpServer: HttpServer): SocketServer {
  const io = new SocketServer(httpServer, {
    cors: {
      origin: process.env.ALLOWED_ORIGINS?.split(",") || "*",
      methods: ["GET", "POST"]
    },
    transports: ["websocket", "polling"]
  });
  
  // Auth middleware
  io.use(async (socket, next) => {
    const token = socket.handshake.auth.token;
    
    if (!token) {
      return next(new Error("Authentication error"));
    }
    
    try {
      const payload = authService.verifyAccessToken(token);
      (socket as AuthenticatedSocket).userId = payload.userId;
      next();
    } catch {
      next(new Error("Invalid token"));
    }
  });
  
  // Subscribe to Redis pub/sub
  const subscriber = redis.duplicate();
  
  io.on("connection", async (socket: Socket) => {
    const userId = (socket as AuthenticatedSocket).userId;
    
    console.log(`User ${userId} connected`);
    
    // Join personal room
    socket.join(`user:${userId}`);
    
    // Mark user as online
    await redis.setex(`online:${userId}`, 300, "1");
    
    // Subscribe to user's notification channel
    await subscriber.subscribe(`user:${userId}:notifications`);
    
    subscriber.on("message", (channel, message) => {
      if (channel === `user:${userId}:notifications`) {
        socket.emit("notification", JSON.parse(message));
      }
    });
    
    // Handle joining conversation rooms
    socket.on("join:conversation", async (conversationId: string) => {
      socket.join(`conversation:${conversationId}`);
      
      // Subscribe to conversation messages
      await subscriber.subscribe(`conversation:${conversationId}`);
    });
    
    socket.on("leave:conversation", (conversationId: string) => {
      socket.leave(`conversation:${conversationId}`);
    });
    
    // Handle typing indicators
    socket.on("typing:start", (conversationId: string) => {
      socket.to(`conversation:${conversationId}`).emit("user:typing", {
        userId,
        conversationId
      });
    });
    
    socket.on("typing:stop", (conversationId: string) => {
      socket.to(`conversation:${conversationId}`).emit("user:stop-typing", {
        userId,
        conversationId
      });
    });
    
    // Handle disconnect
    socket.on("disconnect", async () => {
      console.log(`User ${userId} disconnected`);
      await redis.del(`online:${userId}`);
      
      // Broadcast offline status
      io.emit("user:offline", { userId });
    });
    
    // Broadcast online status
    io.emit("user:online", { userId });
  });
  
  // Forward conversation messages from Redis to WebSocket
  subscriber.on("message", (channel, message) => {
    const match = channel.match(/^conversation:(.+)$/);
    if (match) {
      const conversationId = match[1];
      io.to(`conversation:${conversationId}`).emit("message", JSON.parse(message));
    }
  });
  
  return io;
}
```

---

## Like Service

### src/services/LikeService.ts

```typescript
import { prisma } from "../config/database";
import { notificationService } from "./NotificationService";
import type { Like, ReactionType } from "../types/models";

export class LikeService {
  async likePost(
    userId: string,
    postId: string,
    reactionType: ReactionType = "LIKE"
  ): Promise<{ liked: boolean; likesCount: number }> {
    const existing = await prisma.like.findFirst({
      where: { userId, targetId: postId, targetType: "post" }
    });
    
    if (existing) {
      if (existing.reactionType === reactionType) {
        // Unlike
        await prisma.$transaction([
          prisma.like.delete({ where: { id: existing.id } }),
          prisma.post.update({
            where: { id: postId },
            data: { likesCount: { decrement: 1 } }
          })
        ]);
        
        const post = await prisma.post.findUnique({
          where: { id: postId },
          select: { likesCount: true }
        });
        
        return { liked: false, likesCount: post?.likesCount || 0 };
      } else {
        // Change reaction
        await prisma.like.update({
          where: { id: existing.id },
          data: { reactionType }
        });
        
        const post = await prisma.post.findUnique({
          where: { id: postId },
          select: { likesCount: true }
        });
        
        return { liked: true, likesCount: post?.likesCount || 0 };
      }
    }
    
    // New like
    const [, post] = await prisma.$transaction([
      prisma.like.create({
        data: { userId, targetId: postId, targetType: "post", reactionType }
      }),
      prisma.post.update({
        where: { id: postId },
        data: { likesCount: { increment: 1 } }
      })
    ]);
    
    // Send notification
    const postData = await prisma.post.findUnique({ where: { id: postId } });
    if (postData && postData.authorId !== userId) {
      notificationService.create({
        recipientId: postData.authorId,
        senderId: userId,
        type: "LIKE_POST",
        entityId: postId,
        entityType: "post"
      }).catch(console.error);
    }
    
    return { liked: true, likesCount: post.likesCount };
  }
  
  async likeComment(
    userId: string,
    commentId: string
  ): Promise<{ liked: boolean; likesCount: number }> {
    const existing = await prisma.like.findFirst({
      where: { userId, targetId: commentId, targetType: "comment" }
    });
    
    if (existing) {
      await prisma.$transaction([
        prisma.like.delete({ where: { id: existing.id } }),
        prisma.comment.update({
          where: { id: commentId },
          data: { likesCount: { decrement: 1 } }
        })
      ]);
      
      const comment = await prisma.comment.findUnique({
        where: { id: commentId },
        select: { likesCount: true }
      });
      
      return { liked: false, likesCount: comment?.likesCount || 0 };
    }
    
    const [, comment] = await prisma.$transaction([
      prisma.like.create({
        data: { userId, targetId: commentId, targetType: "comment", reactionType: "LIKE" }
      }),
      prisma.comment.update({
        where: { id: commentId },
        data: { likesCount: { increment: 1 } }
      })
    ]);
    
    return { liked: true, likesCount: comment.likesCount };
  }
  
  async getPostLikes(
    postId: string,
    page: number = 1,
    limit: number = 20
  ) {
    const [total, likes] = await Promise.all([
      prisma.like.count({ where: { targetId: postId, targetType: "post" } }),
      prisma.like.findMany({
        where: { targetId: postId, targetType: "post" },
        orderBy: { createdAt: "desc" },
        skip: (page - 1) * limit,
        take: limit,
        include: {
          user: {
            select: {
              id: true,
              username: true,
              displayName: true,
              avatar: true,
              isVerified: true
            }
          }
        }
      })
    ]);
    
    return { likes, total, page };
  }
  
  async getUserLikedPosts(
    userId: string,
    page: number = 1,
    limit: number = 20
  ) {
    const [total, likes] = await Promise.all([
      prisma.like.count({ where: { userId, targetType: "post" } }),
      prisma.like.findMany({
        where: { userId, targetType: "post" },
        orderBy: { createdAt: "desc" },
        skip: (page - 1) * limit,
        take: limit,
        include: {
          target: true
        }
      })
    ]);
    
    return { posts: likes.map(l => l.target), total, page };
  }
  
  async hasUserLiked(
    userId: string,
    targetId: string,
    targetType: "post" | "comment"
  ): Promise<boolean> {
    const like = await prisma.like.findFirst({
      where: { userId, targetId, targetType }
    });
    return !!like;
  }
}

export const likeService = new LikeService();
```

---

## Comment Service

### src/services/CommentService.ts

```typescript
import { prisma } from "../config/database";
import { notificationService } from "./NotificationService";
import type { Comment } from "../types/models";

export class CommentService {
  async createComment(
    postId: string,
    authorId: string,
    data: {
      content: string;
      parentCommentId?: string;
    }
  ): Promise<Comment> {
    const post = await prisma.post.findUnique({ where: { id: postId } });
    if (!post) throw new Error("ไม่พบโพสต์");
    
    const mentions = this.extractMentions(data.content);
    
    const comment = await prisma.$transaction(async (tx) => {
      const newComment = await tx.comment.create({
        data: {
          postId,
          authorId,
          content: data.content,
          parentCommentId: data.parentCommentId,
          mentions
        },
        include: {
          author: {
            select: {
              id: true,
              username: true,
              displayName: true,
              avatar: true,
              isVerified: true
            }
          }
        }
      });
      
      // Update counts
      await tx.post.update({
        where: { id: postId },
        data: { commentsCount: { increment: 1 } }
      });
      
      if (data.parentCommentId) {
        await tx.comment.update({
          where: { id: data.parentCommentId },
          data: { repliesCount: { increment: 1 } }
        });
      }
      
      return newComment;
    });
    
    // Notifications
    if (post.authorId !== authorId) {
      notificationService.create({
        recipientId: post.authorId,
        senderId: authorId,
        type: "COMMENT_POST",
        entityId: comment.id,
        entityType: "comment"
      }).catch(console.error);
    }
    
    // Notify parent comment author
    if (data.parentCommentId) {
      const parentComment = await prisma.comment.findUnique({
        where: { id: data.parentCommentId }
      });
      
      if (parentComment && parentComment.authorId !== authorId) {
        notificationService.create({
          recipientId: parentComment.authorId,
          senderId: authorId,
          type: "REPLY_COMMENT",
          entityId: comment.id,
          entityType: "comment"
        }).catch(console.error);
      }
    }
    
    // Notify mentioned users
    for (const username of mentions) {
      const user = await prisma.user.findUnique({ where: { username } });
      if (user && user.id !== authorId) {
        notificationService.create({
          recipientId: user.id,
          senderId: authorId,
          type: "MENTION_COMMENT",
          entityId: comment.id,
          entityType: "comment"
        }).catch(console.error);
      }
    }
    
    return comment as unknown as Comment;
  }
  
  async getPostComments(
    postId: string,
    page: number = 1,
    limit: number = 20
  ) {
    const [total, comments] = await Promise.all([
      prisma.comment.count({
        where: { postId, parentCommentId: null }
      }),
      prisma.comment.findMany({
        where: { postId, parentCommentId: null },
        orderBy: [
          { isPinned: "desc" },
          { likesCount: "desc" },
          { createdAt: "asc" }
        ],
        skip: (page - 1) * limit,
        take: limit,
        include: {
          author: {
            select: {
              id: true,
              username: true,
              displayName: true,
              avatar: true,
              isVerified: true
            }
          },
          replies: {
            take: 3,
            orderBy: { createdAt: "asc" },
            include: {
              author: {
                select: {
                  id: true,
                  username: true,
                  displayName: true,
                  avatar: true
                }
              }
            }
          }
        }
      })
    ]);
    
    return { comments, total, page };
  }
  
  async deleteComment(commentId: string, userId: string): Promise<void> {
    const comment = await prisma.comment.findUnique({
      where: { id: commentId }
    });
    
    if (!comment) throw new Error("ไม่พบความคิดเห็น");
    
    const isAuthor = comment.authorId === userId;
    const isAdmin = await prisma.user.findFirst({
      where: { id: userId, role: { in: ["MODERATOR", "ADMIN"] } }
    });
    
    if (!isAuthor && !isAdmin) {
      throw new Error("ไม่มีสิทธิ์ลบความคิดเห็นนี้");
    }
    
    await prisma.$transaction(async (tx) => {
      await tx.comment.delete({ where: { id: commentId } });
      
      await tx.post.update({
        where: { id: comment.postId },
        data: { commentsCount: { decrement: 1 } }
      });
      
      if (comment.parentCommentId) {
        await tx.comment.update({
          where: { id: comment.parentCommentId },
          data: { repliesCount: { decrement: 1 } }
        });
      }
    });
  }
  
  private extractMentions(content: string): string[] {
    const matches = content.match(/@[\w]+/g) || [];
    return [...new Set(matches.map(m => m.slice(1).toLowerCase()))];
  }
}

export const commentService = new CommentService();
```

---

## Main Application

### src/index.ts

```typescript
import express from "express";
import { createServer } from "http";
import cors from "cors";
import helmet from "helmet";
import cookieParser from "cookie-parser";

import { setupWebSocket } from "./websocket";
import { errorHandler } from "./middlewares/errorHandler";
import authRoutes from "./routes/auth";
import userRoutes from "./routes/users";
import postRoutes from "./routes/posts";
import feedRoutes from "./routes/feed";
import messageRoutes from "./routes/messages";

const app = express();
const httpServer = createServer(app);

// Middlewares
app.use(helmet());
app.use(cors({ origin: true, credentials: true }));
app.use(express.json({ limit: "10mb" }));
app.use(express.urlencoded({ extended: true }));
app.use(cookieParser());

// Routes
app.use("/api/auth", authRoutes);
app.use("/api/users", userRoutes);
app.use("/api/posts", postRoutes);
app.use("/api/feed", feedRoutes);
app.use("/api/messages", messageRoutes);

// WebSocket
const io = setupWebSocket(httpServer);
app.set("io", io);

// Health check
app.get("/health", (req, res) => {
  res.json({ status: "ok", timestamp: new Date().toISOString() });
});

app.use(errorHandler);

const PORT = process.env.PORT || 3000;
httpServer.listen(PORT, () => {
  console.log(`Social Media API running on port ${PORT}`);
});

export default app;
```

---

## Redis Configuration

### src/config/redis.ts

```typescript
import Redis from "ioredis";

const REDIS_URL = process.env.REDIS_URL || "redis://localhost:6379";

export const redis = new Redis(REDIS_URL, {
  retryStrategy: (times) => Math.min(times * 50, 2000),
  enableReadyCheck: true,
  maxRetriesPerRequest: 3
});

redis.on("connect", () => console.log("Redis connected"));
redis.on("error", (err) => console.error("Redis error:", err));

export default redis;
```

---

## Prisma Schema

### prisma/schema.prisma

```prisma
model User {
  id                   String    @id @default(cuid())
  username             String    @unique
  email                String    @unique
  passwordHash         String
  displayName          String
  bio                  String?
  avatar               String?
  coverImage           String?
  website              String?
  location             String?
  dateOfBirth          DateTime?
  isVerified           Boolean   @default(false)
  isPrivate            Boolean   @default(false)
  isActive             Boolean   @default(true)
  role                 String    @default("USER")
  followersCount       Int       @default(0)
  followingCount       Int       @default(0)
  postsCount           Int       @default(0)
  notificationSettings Json      @default("{}")
  privacySettings      Json      @default("{}")
  
  posts         Post[]
  comments      Comment[]
  likes         Like[]
  followers     Follow[]      @relation("Following")
  following     Follow[]      @relation("Follower")
  notifications Notification[] @relation("Recipient")
  sentNotifications Notification[] @relation("Sender")
  conversations ConversationParticipant[]
  messages      Message[]
  blocks        UserBlock[]   @relation("Blocker")
  blockedBy     UserBlock[]   @relation("Blocked")
  refreshTokens RefreshToken[]
  
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  
  @@index([username])
  @@index([email])
}

model Post {
  id             String    @id @default(cuid())
  authorId       String
  content        String?
  type           String    @default("POST")
  visibility     String    @default("PUBLIC")
  originalPostId String?
  replyToId      String?
  hashtags       String[]
  mentions       String[]
  location       String?
  isSensitive    Boolean   @default(false)
  isEdited       Boolean   @default(false)
  isPinned       Boolean   @default(false)
  editHistory    Json?
  likesCount     Int       @default(0)
  commentsCount  Int       @default(0)
  repostsCount   Int       @default(0)
  viewsCount     Int       @default(0)
  
  author       User        @relation(fields: [authorId], references: [id], onDelete: Cascade)
  media        PostMedia[]
  likes        Like[]
  comments     Post[]      @relation("PostReplies")
  replyTo      Post?       @relation("PostReplies", fields: [replyToId], references: [id])
  originalPost Post?       @relation("PostReposts", fields: [originalPostId], references: [id])
  reposts      Post[]      @relation("PostReposts")
  
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  
  @@index([authorId])
  @@index([hashtags])
}
```

---

## สรุป

Social Media API ที่สร้างนี้ครอบคลุม:

1. **User Management**: Profiles, privacy settings, blocking
2. **Post System**: Create/edit/delete, media upload, repost, quote
3. **Follow System**: Follow/unfollow, follow requests for private accounts
4. **Feed Algorithm**: Home feed, explore feed, hashtag feed
5. **Notifications**: Real-time notifications via WebSocket/Redis
6. **Messaging**: Direct messages, group conversations, typing indicators
7. **Likes**: Multiple reaction types
8. **Comments**: Nested comments/replies
9. **Real-time**: WebSocket with Socket.io + Redis pub/sub

Technology Stack:
- Node.js + Express + TypeScript
- PostgreSQL + Prisma
- Redis (caching + pub/sub)
- Socket.io (WebSocket)
- JWT (authentication)

---

*จบ Part 58 - Real-world Social Media API*
