# Phase 2 Sprint 4: Video Generation - Action Plan

## Overview

This document provides a comprehensive implementation guide for Phase 2 Sprint 4, focusing on building an automated video generation system using Runway ML for AI video creation, FFmpeg for composition, and ElevenLabs for voice-over narration.

### Sprint Objectives
- [ ] Integrate Runway ML API for AI video generation
- [ ] Implement FFmpeg video composition pipeline
- [ ] Add ElevenLabs voice-over integration
- [ ] Build video generation queue with progress tracking
- [ ] Create video management UI in dashboard

### Prerequisites
- Phase 2 Sprint 1-3 completed
- Runway ML API access
- ElevenLabs API access
- FFmpeg installed on worker nodes
- S3 or cloud storage configured for video assets

---

## 1. Video Generation Module

### 1.1 Module Definition

```typescript
// services/ops-engine/src/modules/video/video.module.ts
import { Module } from '@nestjs/common';
import { BullModule } from '@nestjs/bull';
import { VideoService } from './video.service';
import { VideoController } from './video.controller';
import { VideoProcessor } from './video.processor';
import { RunwayMLService } from './providers/runway-ml.service';
import { ElevenLabsService } from './providers/elevenlabs.service';
import { FFmpegService } from './composition/ffmpeg.service';
import { VideoStorageService } from './storage/video-storage.service';

@Module({
  imports: [
    BullModule.registerQueue({
      name: 'video-generation',
      defaultJobOptions: {
        removeOnComplete: 50,
        removeOnFail: 100,
        attempts: 2,
        backoff: {
          type: 'exponential',
          delay: 10000,
        },
      },
    }),
  ],
  controllers: [VideoController],
  providers: [
    VideoService,
    VideoProcessor,
    RunwayMLService,
    ElevenLabsService,
    FFmpegService,
    VideoStorageService,
  ],
  exports: [VideoService],
})
export class VideoModule {}
```

### 1.2 Video Types and DTOs

```typescript
// services/ops-engine/src/modules/video/dto/video.dto.ts
import { IsString, IsOptional, IsEnum, IsArray, IsNumber, ValidateNested } from 'class-validator';
import { Type } from 'class-transformer';

export enum VideoStyle {
  PRODUCT_SHOWCASE = 'product_showcase',
  LIFESTYLE = 'lifestyle',
  TESTIMONIAL = 'testimonial',
  PROMOTIONAL = 'promotional',
  SOCIAL_AD = 'social_ad',
  STORY = 'story',
}

export enum VideoAspectRatio {
  LANDSCAPE = '16:9',      // YouTube, Website
  PORTRAIT = '9:16',       // TikTok, Reels, Stories
  SQUARE = '1:1',          // Instagram Feed
  VERTICAL = '4:5',        // Instagram Feed (taller)
}

export enum VideoStatus {
  PENDING = 'pending',
  GENERATING_SCENES = 'generating_scenes',
  GENERATING_VOICEOVER = 'generating_voiceover',
  COMPOSING = 'composing',
  PROCESSING = 'processing',
  COMPLETED = 'completed',
  FAILED = 'failed',
}

export class VideoSceneDto {
  @IsString()
  prompt: string;

  @IsNumber()
  duration: number; // seconds

  @IsString()
  @IsOptional()
  imageUrl?: string; // Source image for image-to-video

  @IsString()
  @IsOptional()
  textOverlay?: string;
}

export class CreateVideoDto {
  @IsString()
  storeId: string;

  @IsString()
  @IsOptional()
  productId?: string;

  @IsString()
  title: string;

  @IsEnum(VideoStyle)
  style: VideoStyle;

  @IsEnum(VideoAspectRatio)
  aspectRatio: VideoAspectRatio;

  @IsArray()
  @ValidateNested({ each: true })
  @Type(() => VideoSceneDto)
  scenes: VideoSceneDto[];

  @IsString()
  @IsOptional()
  voiceoverScript?: string;

  @IsString()
  @IsOptional()
  voiceId?: string; // ElevenLabs voice ID

  @IsString()
  @IsOptional()
  musicTrackId?: string;

  @IsNumber()
  @IsOptional()
  musicVolume?: number; // 0-100
}

export class VideoGenerationResult {
  videoId: string;
  status: VideoStatus;
  progress: number;
  videoUrl?: string;
  thumbnailUrl?: string;
  duration?: number;
  error?: string;
}
```

### 1.3 Video Service

```typescript
// services/ops-engine/src/modules/video/video.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { InjectQueue } from '@nestjs/bull';
import { Queue } from 'bull';
import { PrismaService } from '@/shared/database/prisma.service';
import { RedisService } from '@/shared/cache/redis.service';
import { CreateVideoDto, VideoStatus, VideoGenerationResult } from './dto/video.dto';
import { RunwayMLService } from './providers/runway-ml.service';
import { ElevenLabsService } from './providers/elevenlabs.service';
import { FFmpegService } from './composition/ffmpeg.service';
import { VideoStorageService } from './storage/video-storage.service';

@Injectable()
export class VideoService {
  private readonly logger = new Logger(VideoService.name);

  constructor(
    private readonly prisma: PrismaService,
    private readonly redis: RedisService,
    private readonly runwayML: RunwayMLService,
    private readonly elevenLabs: ElevenLabsService,
    private readonly ffmpeg: FFmpegService,
    private readonly storage: VideoStorageService,
    @InjectQueue('video-generation') private readonly videoQueue: Queue,
  ) {}

  async createVideo(dto: CreateVideoDto): Promise<VideoGenerationResult> {
    // Create video record
    const video = await this.prisma.video.create({
      data: {
        storeId: dto.storeId,
        productId: dto.productId,
        title: dto.title,
        style: dto.style,
        aspectRatio: dto.aspectRatio,
        status: VideoStatus.PENDING,
        scenes: dto.scenes as any,
        voiceoverScript: dto.voiceoverScript,
        voiceId: dto.voiceId,
        musicTrackId: dto.musicTrackId,
        musicVolume: dto.musicVolume || 30,
      },
    });

    // Queue the generation job
    await this.videoQueue.add(
      'generate',
      {
        videoId: video.id,
        dto,
      },
      {
        priority: 2,
        timeout: 600000, // 10 minutes
      },
    );

    return {
      videoId: video.id,
      status: VideoStatus.PENDING,
      progress: 0,
    };
  }

  async generateVideo(videoId: string, dto: CreateVideoDto): Promise<void> {
    try {
      // Update status
      await this.updateStatus(videoId, VideoStatus.GENERATING_SCENES, 5);

      // Step 1: Generate video scenes using Runway ML
      const sceneVideos: string[] = [];
      const totalScenes = dto.scenes.length;

      for (let i = 0; i < dto.scenes.length; i++) {
        const scene = dto.scenes[i];
        const progress = 10 + Math.floor((i / totalScenes) * 40);

        await this.updateProgress(videoId, progress);

        const sceneVideo = await this.runwayML.generateVideo({
          prompt: scene.prompt,
          duration: scene.duration,
          aspectRatio: dto.aspectRatio,
          sourceImage: scene.imageUrl,
        });

        sceneVideos.push(sceneVideo.localPath);
      }

      // Step 2: Generate voiceover if script provided
      let voiceoverPath: string | undefined;

      if (dto.voiceoverScript) {
        await this.updateStatus(videoId, VideoStatus.GENERATING_VOICEOVER, 55);

        voiceoverPath = await this.elevenLabs.generateSpeech({
          text: dto.voiceoverScript,
          voiceId: dto.voiceId || 'default',
        });
      }

      // Step 3: Compose final video
      await this.updateStatus(videoId, VideoStatus.COMPOSING, 65);

      const composedVideo = await this.ffmpeg.composeVideo({
        scenes: sceneVideos,
        voiceover: voiceoverPath,
        musicTrack: dto.musicTrackId ? await this.getMusicTrackPath(dto.musicTrackId) : undefined,
        musicVolume: dto.musicVolume || 30,
        aspectRatio: dto.aspectRatio,
        textOverlays: dto.scenes.map(s => s.textOverlay).filter(Boolean),
      });

      await this.updateProgress(videoId, 85);

      // Step 4: Generate thumbnail
      const thumbnailPath = await this.ffmpeg.extractThumbnail(composedVideo, 2);

      // Step 5: Upload to storage
      await this.updateStatus(videoId, VideoStatus.PROCESSING, 90);

      const [videoUrl, thumbnailUrl] = await Promise.all([
        this.storage.uploadVideo(composedVideo, videoId),
        this.storage.uploadThumbnail(thumbnailPath, videoId),
      ]);

      // Get video duration
      const duration = await this.ffmpeg.getVideoDuration(composedVideo);

      // Update final status
      await this.prisma.video.update({
        where: { id: videoId },
        data: {
          status: VideoStatus.COMPLETED,
          videoUrl,
          thumbnailUrl,
          duration,
          completedAt: new Date(),
        },
      });

      await this.updateProgress(videoId, 100);

      // Cleanup temp files
      await this.cleanup([...sceneVideos, voiceoverPath, composedVideo, thumbnailPath]);

      this.logger.log(`Video generation completed: ${videoId}`);
    } catch (error) {
      this.logger.error(`Video generation failed: ${videoId}`, error);

      await this.prisma.video.update({
        where: { id: videoId },
        data: {
          status: VideoStatus.FAILED,
          error: error.message,
        },
      });

      throw error;
    }
  }

  async getVideoStatus(videoId: string): Promise<VideoGenerationResult> {
    const video = await this.prisma.video.findUnique({
      where: { id: videoId },
    });

    if (!video) {
      throw new Error(`Video not found: ${videoId}`);
    }

    const progress = await this.getProgress(videoId);

    return {
      videoId: video.id,
      status: video.status as VideoStatus,
      progress,
      videoUrl: video.videoUrl,
      thumbnailUrl: video.thumbnailUrl,
      duration: video.duration,
      error: video.error,
    };
  }

  async getStoreVideos(
    storeId: string,
    options: {
      status?: VideoStatus;
      productId?: string;
      limit?: number;
      offset?: number;
    } = {},
  ) {
    const where: any = { storeId };

    if (options.status) where.status = options.status;
    if (options.productId) where.productId = options.productId;

    const [videos, total] = await Promise.all([
      this.prisma.video.findMany({
        where,
        orderBy: { createdAt: 'desc' },
        take: options.limit || 20,
        skip: options.offset || 0,
        include: {
          product: {
            select: {
              id: true,
              title: true,
              images: true,
            },
          },
        },
      }),
      this.prisma.video.count({ where }),
    ]);

    return {
      videos,
      total,
      hasMore: (options.offset || 0) + videos.length < total,
    };
  }

  async deleteVideo(videoId: string): Promise<void> {
    const video = await this.prisma.video.findUnique({
      where: { id: videoId },
    });

    if (!video) {
      throw new Error(`Video not found: ${videoId}`);
    }

    // Delete from storage
    if (video.videoUrl) {
      await this.storage.deleteVideo(video.videoUrl);
    }
    if (video.thumbnailUrl) {
      await this.storage.deleteThumbnail(video.thumbnailUrl);
    }

    // Delete record
    await this.prisma.video.delete({
      where: { id: videoId },
    });
  }

  async generateProductVideo(
    productId: string,
    style: VideoStyle = VideoStyle.PRODUCT_SHOWCASE,
  ): Promise<VideoGenerationResult> {
    const product = await this.prisma.product.findUnique({
      where: { id: productId },
      include: {
        store: true,
      },
    });

    if (!product) {
      throw new Error(`Product not found: ${productId}`);
    }

    // Generate scenes from product data
    const scenes = await this.generateProductScenes(product, style);

    // Generate voiceover script
    const voiceoverScript = await this.generateVoiceoverScript(product, style);

    return this.createVideo({
      storeId: product.storeId,
      productId: product.id,
      title: `${product.title} - ${style}`,
      style,
      aspectRatio: VideoAspectRatio.PORTRAIT,
      scenes,
      voiceoverScript,
    });
  }

  private async generateProductScenes(product: any, style: VideoStyle) {
    const images = product.images || [];
    const scenes = [];

    // Opening scene
    scenes.push({
      prompt: `Professional product showcase of ${product.title}, elegant lighting, ${product.store?.niche || 'retail'} style, cinematic`,
      duration: 3,
      imageUrl: images[0]?.url,
      textOverlay: product.title,
    });

    // Feature scenes from additional images
    for (let i = 1; i < Math.min(images.length, 4); i++) {
      scenes.push({
        prompt: `${product.title} product detail shot, highlighting features, professional photography style`,
        duration: 2,
        imageUrl: images[i]?.url,
      });
    }

    // Call to action scene
    scenes.push({
      prompt: `Clean minimalist background with subtle motion, perfect for product overlay and call to action`,
      duration: 3,
      textOverlay: `Shop Now - $${product.sellingPrice}`,
    });

    return scenes;
  }

  private async generateVoiceoverScript(product: any, style: VideoStyle): Promise<string> {
    // Simple script generation - could be enhanced with AI
    const description = product.description?.substring(0, 200) || '';

    return `Introducing ${product.title}. ${description}. Available now for just $${product.sellingPrice}. Shop today and transform your ${product.store?.niche || 'lifestyle'}.`;
  }

  private async updateStatus(videoId: string, status: VideoStatus, progress: number): Promise<void> {
    await Promise.all([
      this.prisma.video.update({
        where: { id: videoId },
        data: { status },
      }),
      this.updateProgress(videoId, progress),
    ]);
  }

  private async updateProgress(videoId: string, progress: number): Promise<void> {
    await this.redis.set(`video:progress:${videoId}`, progress.toString(), 3600);
  }

  private async getProgress(videoId: string): Promise<number> {
    const progress = await this.redis.get<string>(`video:progress:${videoId}`);
    return progress ? parseInt(progress) : 0;
  }

  private async getMusicTrackPath(trackId: string): Promise<string> {
    // Fetch music track from storage or predefined library
    return `/assets/music/${trackId}.mp3`;
  }

  private async cleanup(paths: (string | undefined)[]): Promise<void> {
    const fs = await import('fs/promises');

    for (const path of paths) {
      if (path) {
        try {
          await fs.unlink(path);
        } catch (error) {
          this.logger.warn(`Failed to cleanup: ${path}`);
        }
      }
    }
  }
}
```

---

## 2. Runway ML Integration

### 2.1 Runway ML Service

```typescript
// services/ops-engine/src/modules/video/providers/runway-ml.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import axios, { AxiosInstance } from 'axios';
import * as fs from 'fs/promises';
import * as path from 'path';
import { v4 as uuid } from 'uuid';

export interface RunwayVideoRequest {
  prompt: string;
  duration: number;
  aspectRatio: string;
  sourceImage?: string;
}

export interface RunwayVideoResult {
  taskId: string;
  videoUrl: string;
  localPath: string;
}

@Injectable()
export class RunwayMLService {
  private readonly logger = new Logger(RunwayMLService.name);
  private readonly client: AxiosInstance;
  private readonly apiKey: string;
  private readonly tempDir: string;

  constructor(private readonly configService: ConfigService) {
    this.apiKey = this.configService.get('RUNWAY_API_KEY');
    this.tempDir = this.configService.get('TEMP_DIR', '/tmp/videos');

    this.client = axios.create({
      baseURL: 'https://api.runwayml.com/v1',
      headers: {
        'Authorization': `Bearer ${this.apiKey}`,
        'Content-Type': 'application/json',
        'X-Runway-Version': '2024-09-13',
      },
      timeout: 300000, // 5 minutes
    });
  }

  async generateVideo(request: RunwayVideoRequest): Promise<RunwayVideoResult> {
    try {
      // Determine which model to use
      const model = request.sourceImage ? 'gen3a_turbo' : 'gen3a_turbo';

      // Prepare the request payload
      const payload: any = {
        promptText: this.enhancePrompt(request.prompt),
        model,
        duration: Math.min(request.duration, 10), // Runway max is 10 seconds
        ratio: this.mapAspectRatio(request.aspectRatio),
        watermark: false,
      };

      // Add source image for image-to-video
      if (request.sourceImage) {
        payload.promptImage = request.sourceImage;
      }

      // Create generation task
      const createResponse = await this.client.post('/image_to_video', payload);
      const taskId = createResponse.data.id;

      this.logger.log(`Runway task created: ${taskId}`);

      // Poll for completion
      const result = await this.pollForCompletion(taskId);

      // Download the video
      const localPath = await this.downloadVideo(result.output[0], taskId);

      return {
        taskId,
        videoUrl: result.output[0],
        localPath,
      };
    } catch (error) {
      this.logger.error('Runway ML generation failed:', error.response?.data || error.message);
      throw new Error(`Video generation failed: ${error.message}`);
    }
  }

  private async pollForCompletion(taskId: string, maxAttempts: number = 60): Promise<any> {
    for (let attempt = 0; attempt < maxAttempts; attempt++) {
      const response = await this.client.get(`/tasks/${taskId}`);
      const status = response.data.status;

      if (status === 'SUCCEEDED') {
        return response.data;
      }

      if (status === 'FAILED') {
        throw new Error(`Generation failed: ${response.data.error || 'Unknown error'}`);
      }

      // Wait before next poll (5 seconds)
      await this.delay(5000);
    }

    throw new Error('Generation timed out');
  }

  private async downloadVideo(url: string, taskId: string): Promise<string> {
    const response = await axios.get(url, { responseType: 'arraybuffer' });

    await fs.mkdir(this.tempDir, { recursive: true });
    const localPath = path.join(this.tempDir, `${taskId}_${uuid()}.mp4`);

    await fs.writeFile(localPath, response.data);

    return localPath;
  }

  private enhancePrompt(prompt: string): string {
    // Add quality modifiers to the prompt
    const qualityModifiers = 'high quality, professional, smooth motion, cinematic';
    return `${prompt}, ${qualityModifiers}`;
  }

  private mapAspectRatio(ratio: string): string {
    const mapping: Record<string, string> = {
      '16:9': '16:9',
      '9:16': '9:16',
      '1:1': '1:1',
      '4:5': '9:16', // Map to closest supported ratio
    };

    return mapping[ratio] || '16:9';
  }

  private delay(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}
```

---

## 3. ElevenLabs Voice-over Integration

### 3.1 ElevenLabs Service

```typescript
// services/ops-engine/src/modules/video/providers/elevenlabs.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import axios, { AxiosInstance } from 'axios';
import * as fs from 'fs/promises';
import * as path from 'path';
import { v4 as uuid } from 'uuid';

export interface SpeechRequest {
  text: string;
  voiceId: string;
  modelId?: string;
  stability?: number;
  similarityBoost?: number;
}

@Injectable()
export class ElevenLabsService {
  private readonly logger = new Logger(ElevenLabsService.name);
  private readonly client: AxiosInstance;
  private readonly tempDir: string;

  // Default voice IDs
  private readonly defaultVoices: Record<string, string> = {
    default: 'pNInz6obpgDQGcFmaJgB', // Adam - versatile male voice
    female: 'EXAVITQu4vr4xnSDxMaL',   // Bella - warm female voice
    male: 'VR6AewLTigWG4xSOukaG',     // Arnold - confident male voice
    young: 'jBpfuIE2acCO8z3wKNLl',    // Gigi - youthful female voice
  };

  constructor(private readonly configService: ConfigService) {
    const apiKey = this.configService.get('ELEVENLABS_API_KEY');
    this.tempDir = this.configService.get('TEMP_DIR', '/tmp/audio');

    this.client = axios.create({
      baseURL: 'https://api.elevenlabs.io/v1',
      headers: {
        'xi-api-key': apiKey,
        'Content-Type': 'application/json',
      },
      timeout: 60000,
    });
  }

  async generateSpeech(request: SpeechRequest): Promise<string> {
    try {
      const voiceId = this.resolveVoiceId(request.voiceId);

      const response = await this.client.post(
        `/text-to-speech/${voiceId}`,
        {
          text: request.text,
          model_id: request.modelId || 'eleven_monolingual_v1',
          voice_settings: {
            stability: request.stability || 0.5,
            similarity_boost: request.similarityBoost || 0.75,
          },
        },
        {
          responseType: 'arraybuffer',
        },
      );

      // Save to temp file
      await fs.mkdir(this.tempDir, { recursive: true });
      const localPath = path.join(this.tempDir, `voiceover_${uuid()}.mp3`);

      await fs.writeFile(localPath, response.data);

      this.logger.log(`Voice-over generated: ${localPath}`);

      return localPath;
    } catch (error) {
      this.logger.error('ElevenLabs generation failed:', error.response?.data || error.message);
      throw new Error(`Voice-over generation failed: ${error.message}`);
    }
  }

  async getAvailableVoices(): Promise<Array<{ id: string; name: string; category: string }>> {
    try {
      const response = await this.client.get('/voices');

      return response.data.voices.map((voice: any) => ({
        id: voice.voice_id,
        name: voice.name,
        category: voice.category,
        preview: voice.preview_url,
      }));
    } catch (error) {
      this.logger.error('Failed to fetch voices:', error);
      throw error;
    }
  }

  async getVoicePreview(voiceId: string): Promise<string> {
    const voices = await this.getAvailableVoices();
    const voice = voices.find(v => v.id === voiceId);

    if (!voice) {
      throw new Error(`Voice not found: ${voiceId}`);
    }

    return (voice as any).preview;
  }

  private resolveVoiceId(voiceId: string): string {
    // Check if it's a preset name
    if (this.defaultVoices[voiceId]) {
      return this.defaultVoices[voiceId];
    }

    // Otherwise, assume it's a direct voice ID
    return voiceId;
  }
}
```

---

## 4. FFmpeg Video Composition

### 4.1 FFmpeg Service

```typescript
// services/ops-engine/src/modules/video/composition/ffmpeg.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import * as ffmpeg from 'fluent-ffmpeg';
import * as path from 'path';
import * as fs from 'fs/promises';
import { v4 as uuid } from 'uuid';

export interface ComposeVideoOptions {
  scenes: string[];
  voiceover?: string;
  musicTrack?: string;
  musicVolume: number;
  aspectRatio: string;
  textOverlays?: (string | undefined)[];
}

@Injectable()
export class FFmpegService {
  private readonly logger = new Logger(FFmpegService.name);
  private readonly tempDir: string;
  private readonly assetsDir: string;

  constructor(private readonly configService: ConfigService) {
    this.tempDir = this.configService.get('TEMP_DIR', '/tmp/videos');
    this.assetsDir = this.configService.get('ASSETS_DIR', '/assets');

    // Set FFmpeg path if provided
    const ffmpegPath = this.configService.get('FFMPEG_PATH');
    if (ffmpegPath) {
      ffmpeg.setFfmpegPath(ffmpegPath);
    }
  }

  async composeVideo(options: ComposeVideoOptions): Promise<string> {
    await fs.mkdir(this.tempDir, { recursive: true });

    // Step 1: Concatenate scene videos
    const concatenatedPath = await this.concatenateVideos(options.scenes);

    // Step 2: Add text overlays if provided
    let processedPath = concatenatedPath;
    if (options.textOverlays?.some(t => t)) {
      processedPath = await this.addTextOverlays(
        processedPath,
        options.textOverlays,
        options.scenes.length,
      );
    }

    // Step 3: Mix audio tracks
    const outputPath = path.join(this.tempDir, `final_${uuid()}.mp4`);

    await this.mixAudio({
      videoPath: processedPath,
      voiceoverPath: options.voiceover,
      musicPath: options.musicTrack,
      musicVolume: options.musicVolume,
      outputPath,
    });

    // Cleanup intermediate files
    if (concatenatedPath !== processedPath) {
      await this.safeUnlink(concatenatedPath);
    }
    if (processedPath !== outputPath) {
      await this.safeUnlink(processedPath);
    }

    return outputPath;
  }

  async concatenateVideos(videoPaths: string[]): Promise<string> {
    const outputPath = path.join(this.tempDir, `concat_${uuid()}.mp4`);
    const listPath = path.join(this.tempDir, `concat_list_${uuid()}.txt`);

    // Create concat list file
    const listContent = videoPaths.map(p => `file '${p}'`).join('\n');
    await fs.writeFile(listPath, listContent);

    return new Promise((resolve, reject) => {
      ffmpeg()
        .input(listPath)
        .inputOptions(['-f concat', '-safe 0'])
        .outputOptions(['-c copy'])
        .output(outputPath)
        .on('end', async () => {
          await this.safeUnlink(listPath);
          resolve(outputPath);
        })
        .on('error', (err) => {
          this.logger.error('Concatenation failed:', err);
          reject(err);
        })
        .run();
    });
  }

  async addTextOverlays(
    videoPath: string,
    overlays: (string | undefined)[],
    sceneCount: number,
  ): Promise<string> {
    const outputPath = path.join(this.tempDir, `overlay_${uuid()}.mp4`);

    // Build drawtext filters for each scene
    const filters: string[] = [];
    let currentTime = 0;
    const sceneDuration = await this.getVideoDuration(videoPath) / sceneCount;

    for (let i = 0; i < overlays.length; i++) {
      const text = overlays[i];
      if (text) {
        const startTime = currentTime;
        const endTime = currentTime + sceneDuration;

        filters.push(
          `drawtext=text='${this.escapeText(text)}':` +
          `fontfile=/usr/share/fonts/truetype/dejavu/DejaVuSans-Bold.ttf:` +
          `fontsize=48:fontcolor=white:` +
          `borderw=2:bordercolor=black:` +
          `x=(w-text_w)/2:y=h-100:` +
          `enable='between(t,${startTime},${endTime})'`,
        );
      }
      currentTime += sceneDuration;
    }

    if (filters.length === 0) {
      return videoPath;
    }

    return new Promise((resolve, reject) => {
      ffmpeg(videoPath)
        .videoFilters(filters.join(','))
        .outputOptions(['-c:a copy'])
        .output(outputPath)
        .on('end', () => resolve(outputPath))
        .on('error', (err) => {
          this.logger.error('Text overlay failed:', err);
          reject(err);
        })
        .run();
    });
  }

  async mixAudio(options: {
    videoPath: string;
    voiceoverPath?: string;
    musicPath?: string;
    musicVolume: number;
    outputPath: string;
  }): Promise<void> {
    return new Promise((resolve, reject) => {
      let command = ffmpeg(options.videoPath);
      const filterComplex: string[] = [];
      let audioInputIndex = 1;

      // Add voiceover input
      if (options.voiceoverPath) {
        command = command.input(options.voiceoverPath);
        filterComplex.push(`[${audioInputIndex}:a]volume=1.0[voice]`);
        audioInputIndex++;
      }

      // Add music input
      if (options.musicPath) {
        command = command.input(options.musicPath);
        const volumeNormalized = options.musicVolume / 100;
        filterComplex.push(`[${audioInputIndex}:a]volume=${volumeNormalized}[music]`);
        audioInputIndex++;
      }

      // Build audio mix
      if (filterComplex.length > 0) {
        const audioSources: string[] = [];
        if (options.voiceoverPath) audioSources.push('[voice]');
        if (options.musicPath) audioSources.push('[music]');

        if (audioSources.length === 2) {
          filterComplex.push(`${audioSources.join('')}amix=inputs=2:duration=first[aout]`);
        } else if (audioSources.length === 1) {
          filterComplex.push(`${audioSources[0]}acopy[aout]`);
        }

        command = command
          .complexFilter(filterComplex.join(';'))
          .outputOptions([
            '-map 0:v',
            '-map [aout]',
            '-c:v copy',
            '-c:a aac',
            '-shortest',
          ]);
      } else {
        command = command.outputOptions(['-c copy']);
      }

      command
        .output(options.outputPath)
        .on('end', () => resolve())
        .on('error', (err) => {
          this.logger.error('Audio mix failed:', err);
          reject(err);
        })
        .run();
    });
  }

  async extractThumbnail(videoPath: string, timestamp: number = 1): Promise<string> {
    const outputPath = path.join(this.tempDir, `thumb_${uuid()}.jpg`);

    return new Promise((resolve, reject) => {
      ffmpeg(videoPath)
        .screenshots({
          timestamps: [timestamp],
          filename: path.basename(outputPath),
          folder: path.dirname(outputPath),
          size: '1280x720',
        })
        .on('end', () => resolve(outputPath))
        .on('error', (err) => {
          this.logger.error('Thumbnail extraction failed:', err);
          reject(err);
        });
    });
  }

  async getVideoDuration(videoPath: string): Promise<number> {
    return new Promise((resolve, reject) => {
      ffmpeg.ffprobe(videoPath, (err, metadata) => {
        if (err) {
          reject(err);
          return;
        }

        resolve(metadata.format.duration || 0);
      });
    });
  }

  async resizeVideo(
    inputPath: string,
    width: number,
    height: number,
  ): Promise<string> {
    const outputPath = path.join(this.tempDir, `resized_${uuid()}.mp4`);

    return new Promise((resolve, reject) => {
      ffmpeg(inputPath)
        .size(`${width}x${height}`)
        .outputOptions(['-c:a copy'])
        .output(outputPath)
        .on('end', () => resolve(outputPath))
        .on('error', reject)
        .run();
    });
  }

  private escapeText(text: string): string {
    return text
      .replace(/'/g, "\\'")
      .replace(/:/g, '\\:')
      .replace(/\\/g, '\\\\');
  }

  private async safeUnlink(filePath: string): Promise<void> {
    try {
      await fs.unlink(filePath);
    } catch (error) {
      this.logger.warn(`Failed to delete temp file: ${filePath}`);
    }
  }
}
```

---

## 5. Video Storage Service

```typescript
// services/ops-engine/src/modules/video/storage/video-storage.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { S3Client, PutObjectCommand, DeleteObjectCommand } from '@aws-sdk/client-s3';
import * as fs from 'fs/promises';
import * as path from 'path';

@Injectable()
export class VideoStorageService {
  private readonly logger = new Logger(VideoStorageService.name);
  private readonly s3Client: S3Client;
  private readonly bucket: string;
  private readonly cdnDomain: string;

  constructor(private readonly configService: ConfigService) {
    this.s3Client = new S3Client({
      region: this.configService.get('AWS_REGION', 'us-east-1'),
      credentials: {
        accessKeyId: this.configService.get('AWS_ACCESS_KEY_ID'),
        secretAccessKey: this.configService.get('AWS_SECRET_ACCESS_KEY'),
      },
    });

    this.bucket = this.configService.get('S3_VIDEO_BUCKET', 'zyenta-videos');
    this.cdnDomain = this.configService.get('CDN_DOMAIN', `${this.bucket}.s3.amazonaws.com`);
  }

  async uploadVideo(localPath: string, videoId: string): Promise<string> {
    const key = `videos/${videoId}/video.mp4`;

    const fileContent = await fs.readFile(localPath);

    await this.s3Client.send(
      new PutObjectCommand({
        Bucket: this.bucket,
        Key: key,
        Body: fileContent,
        ContentType: 'video/mp4',
        CacheControl: 'max-age=31536000',
      }),
    );

    this.logger.log(`Uploaded video: ${key}`);

    return `https://${this.cdnDomain}/${key}`;
  }

  async uploadThumbnail(localPath: string, videoId: string): Promise<string> {
    const key = `videos/${videoId}/thumbnail.jpg`;

    const fileContent = await fs.readFile(localPath);

    await this.s3Client.send(
      new PutObjectCommand({
        Bucket: this.bucket,
        Key: key,
        Body: fileContent,
        ContentType: 'image/jpeg',
        CacheControl: 'max-age=31536000',
      }),
    );

    this.logger.log(`Uploaded thumbnail: ${key}`);

    return `https://${this.cdnDomain}/${key}`;
  }

  async deleteVideo(videoUrl: string): Promise<void> {
    const key = this.extractKeyFromUrl(videoUrl);

    if (key) {
      await this.s3Client.send(
        new DeleteObjectCommand({
          Bucket: this.bucket,
          Key: key,
        }),
      );

      this.logger.log(`Deleted video: ${key}`);
    }
  }

  async deleteThumbnail(thumbnailUrl: string): Promise<void> {
    const key = this.extractKeyFromUrl(thumbnailUrl);

    if (key) {
      await this.s3Client.send(
        new DeleteObjectCommand({
          Bucket: this.bucket,
          Key: key,
        }),
      );

      this.logger.log(`Deleted thumbnail: ${key}`);
    }
  }

  private extractKeyFromUrl(url: string): string | null {
    try {
      const urlObj = new URL(url);
      return urlObj.pathname.slice(1); // Remove leading slash
    } catch {
      return null;
    }
  }
}
```

---

## 6. Video Processor (Queue Worker)

```typescript
// services/ops-engine/src/modules/video/video.processor.ts
import { Process, Processor, OnQueueFailed, OnQueueCompleted } from '@nestjs/bull';
import { Logger } from '@nestjs/common';
import { Job } from 'bull';
import { VideoService } from './video.service';
import { CreateVideoDto } from './dto/video.dto';

@Processor('video-generation')
export class VideoProcessor {
  private readonly logger = new Logger(VideoProcessor.name);

  constructor(private readonly videoService: VideoService) {}

  @Process('generate')
  async handleGenerate(job: Job<{ videoId: string; dto: CreateVideoDto }>) {
    this.logger.log(`Processing video generation job: ${job.id}`);

    const { videoId, dto } = job.data;

    await this.videoService.generateVideo(videoId, dto);

    return { videoId, status: 'completed' };
  }

  @OnQueueCompleted()
  onCompleted(job: Job, result: any) {
    this.logger.log(`Video generation job ${job.id} completed: ${result.videoId}`);
  }

  @OnQueueFailed()
  onFailed(job: Job, error: Error) {
    this.logger.error(`Video generation job ${job.id} failed:`, error);
  }
}
```

---

## 7. Video Controller

```typescript
// services/ops-engine/src/modules/video/video.controller.ts
import {
  Controller,
  Get,
  Post,
  Delete,
  Param,
  Body,
  Query,
  HttpCode,
  HttpStatus,
} from '@nestjs/common';
import { VideoService } from './video.service';
import { CreateVideoDto, VideoStyle } from './dto/video.dto';
import { ElevenLabsService } from './providers/elevenlabs.service';

@Controller('videos')
export class VideoController {
  constructor(
    private readonly videoService: VideoService,
    private readonly elevenLabs: ElevenLabsService,
  ) {}

  @Post()
  @HttpCode(HttpStatus.ACCEPTED)
  async createVideo(@Body() dto: CreateVideoDto) {
    return this.videoService.createVideo(dto);
  }

  @Get(':videoId')
  async getVideo(@Param('videoId') videoId: string) {
    return this.videoService.getVideoStatus(videoId);
  }

  @Get(':videoId/status')
  async getVideoStatus(@Param('videoId') videoId: string) {
    return this.videoService.getVideoStatus(videoId);
  }

  @Delete(':videoId')
  @HttpCode(HttpStatus.NO_CONTENT)
  async deleteVideo(@Param('videoId') videoId: string) {
    await this.videoService.deleteVideo(videoId);
  }

  @Get('store/:storeId')
  async getStoreVideos(
    @Param('storeId') storeId: string,
    @Query('status') status?: string,
    @Query('productId') productId?: string,
    @Query('limit') limit?: string,
    @Query('offset') offset?: string,
  ) {
    return this.videoService.getStoreVideos(storeId, {
      status: status as any,
      productId,
      limit: limit ? parseInt(limit) : undefined,
      offset: offset ? parseInt(offset) : undefined,
    });
  }

  @Post('product/:productId')
  @HttpCode(HttpStatus.ACCEPTED)
  async generateProductVideo(
    @Param('productId') productId: string,
    @Query('style') style?: VideoStyle,
  ) {
    return this.videoService.generateProductVideo(productId, style);
  }

  @Get('voices/available')
  async getAvailableVoices() {
    return this.elevenLabs.getAvailableVoices();
  }
}
```

---

## 8. Database Schema Updates

```prisma
// packages/database/prisma/schema.prisma (additions)

model Video {
  id              String    @id @default(cuid())
  storeId         String
  store           Store     @relation(fields: [storeId], references: [id], onDelete: Cascade)
  productId       String?
  product         Product?  @relation(fields: [productId], references: [id], onDelete: SetNull)

  title           String
  style           String    // product_showcase, lifestyle, etc.
  aspectRatio     String    // 16:9, 9:16, 1:1
  status          String    @default("pending")

  // Generation config
  scenes          Json
  voiceoverScript String?   @db.Text
  voiceId         String?
  musicTrackId    String?
  musicVolume     Int       @default(30)

  // Output
  videoUrl        String?
  thumbnailUrl    String?
  duration        Float?

  error           String?   @db.Text

  completedAt     DateTime?
  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt

  @@index([storeId])
  @@index([productId])
  @@index([status])
}

model MusicTrack {
  id          String    @id @default(cuid())
  name        String
  category    String    // upbeat, calm, corporate, etc.
  duration    Float
  fileUrl     String
  previewUrl  String?

  createdAt   DateTime  @default(now())

  @@index([category])
}
```

---

## 9. Testing

### 9.1 Video Service Tests

```typescript
// services/ops-engine/src/modules/video/__tests__/video.service.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { VideoService } from '../video.service';
import { VideoStyle, VideoAspectRatio } from '../dto/video.dto';

describe('VideoService', () => {
  let service: VideoService;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        VideoService,
        // ... mock providers
      ],
    }).compile();

    service = module.get<VideoService>(VideoService);
  });

  describe('createVideo', () => {
    it('should create a video record and queue job', async () => {
      const dto = {
        storeId: 'store-1',
        title: 'Test Video',
        style: VideoStyle.PRODUCT_SHOWCASE,
        aspectRatio: VideoAspectRatio.PORTRAIT,
        scenes: [
          { prompt: 'Test scene', duration: 3 },
        ],
      };

      const result = await service.createVideo(dto);

      expect(result.videoId).toBeDefined();
      expect(result.status).toBe('pending');
      expect(result.progress).toBe(0);
    });
  });

  describe('generateProductScenes', () => {
    it('should generate scenes from product data', async () => {
      const product = {
        title: 'Test Product',
        description: 'A great product',
        sellingPrice: 29.99,
        images: [{ url: 'http://example.com/image.jpg' }],
        store: { niche: 'fashion' },
      };

      const scenes = await service['generateProductScenes'](product, VideoStyle.PRODUCT_SHOWCASE);

      expect(scenes.length).toBeGreaterThan(0);
      expect(scenes[0].prompt).toContain('Test Product');
    });
  });
});
```

---

## Verification Checklist

### Runway ML Integration
- [ ] API authentication works
- [ ] Text-to-video generation works
- [ ] Image-to-video generation works
- [ ] Polling for completion works
- [ ] Video download and storage works

### ElevenLabs Integration
- [ ] Voice list retrieval works
- [ ] Text-to-speech generation works
- [ ] Audio file is saved correctly
- [ ] Different voices work

### FFmpeg Composition
- [ ] Video concatenation works
- [ ] Text overlays render correctly
- [ ] Audio mixing works
- [ ] Thumbnail extraction works
- [ ] Duration detection works

### Video Service
- [ ] Video creation queues job
- [ ] Progress tracking works
- [ ] Status updates correctly
- [ ] Completed videos have URLs
- [ ] Failed videos have errors logged

### Storage
- [ ] Videos upload to S3
- [ ] Thumbnails upload correctly
- [ ] CDN URLs are correct
- [ ] Deletion works

---

## Next Steps

After completing Sprint 4, proceed to:

**Sprint 5: Integration & Testing**
- End-to-end fulfillment testing
- Inventory sync stress testing
- Video generation quality assurance
- Performance optimization
- Documentation and training

---

*Phase 2 Sprint 4 establishes the automated video generation pipeline using Runway ML for AI video creation, ElevenLabs for voice-over narration, and FFmpeg for professional video composition.*
