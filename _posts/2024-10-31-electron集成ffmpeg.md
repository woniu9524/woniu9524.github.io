---
layout: post
title: electron集成ffmpeg
tags: [electron, ffmpeg,]
---
### 依赖安装

在electron项目中集成ffmpeg，需要安装@ffmpeg-installer/ffmpeg和fluent-ffmpeg

```
npm install @ffmpeg-installer/ffmpeg fluent-ffmpeg --save
```
其中@ffmpeg-installer会预编译不同平台的二进制文件，通过 npm 包管理和分发这些二进制文件，运行时根据平台选择正确的二进制文件，并且能正确处理文件路径和权限。我们也可以参照类似的方法打包任意命令行工具。而fluent-ffmpeg 是一个 Node.js 的 FFmpeg 命令行封装库。

### 使用

这里我使用的egg框架，使用的时候需要注意编译的问题。在开发环境中，使用ffmpegInstaller.path获取就可以了。编译后ffmpeg在resources\app.asar.unpacked目录下，所以我的需对代码做些修改。如果不是egg，或许情况稍有不同，具体看目录结构和报错就好了。

```javascript

// services/video.js
'use strict';

const { Service } = require('ee-core');
const ffmpeg = require('fluent-ffmpeg');
const ffmpegInstaller = require('@ffmpeg-installer/ffmpeg');
const ffprobeInstaller = require('@ffprobe-installer/ffprobe');
const Log = require('ee-core/log');

class VideoService extends Service {
    constructor(ctx) {
        super(ctx);
        this.initializeFfmpeg();
    }

    initializeFfmpeg() {
        try {
            // 获取正确的 ffmpeg 路径
            const getFfmpegPath = () => {
                if (process.env.EE_SERVER_ENV === 'local') {
                    return ffmpegInstaller.path;
                }
                // 生产环境
                const { app } = require('electron');
                const path = require('path');

                // 方案1：如果使用 asarUnpack
                const appPath = app.getAppPath();
                const unpackedPath = appPath.replace('app.asar', 'app.asar.unpacked');
                return path.join(
                    unpackedPath,
                    'node_modules',
                    '@ffmpeg-installer',
                    'win32-x64',
                    'ffmpeg.exe'
                );

            };

            // 获取正确的 ffprobe 路径
            const getFfprobePath = () => {
                if (process.env.EE_SERVER_ENV === 'local') {
                    return ffprobeInstaller.path;
                }
                const { app } = require('electron');
                const path = require('path');

                // 方案1：如果使用 asarUnpack
                const appPath = app.getAppPath();
                const unpackedPath = appPath.replace('app.asar', 'app.asar.unpacked');
                return path.join(
                    unpackedPath,
                    'node_modules',
                    '@ffprobe-installer',
                    'win32-x64',
                    'ffprobe.exe'
                );
            };

            const ffmpegPath = getFfmpegPath();
            const ffprobePath = getFfprobePath();

            Log.info('FFmpeg path:', ffmpegPath);
            Log.info('FFprobe path:', ffprobePath);

            // 检查文件是否存在
            const fs = require('fs');
            if (!fs.existsSync(ffmpegPath)) {
                throw new Error(`FFmpeg executable not found at: ${ffmpegPath}`);
            }
            if (!fs.existsSync(ffprobePath)) {
                throw new Error(`FFprobe executable not found at: ${ffprobePath}`);
            }

            // 设置路径
            ffmpeg.setFfmpegPath(ffmpegPath);
            ffmpeg.setFfprobePath(ffprobePath);

            // 测试 FFmpeg 是否可用
            ffmpeg.getAvailableFormats((err, formats) => {
                if (err) {
                    Log.error('FFmpeg test failed:', err);
                    throw err;
                }
                Log.info('FFmpeg initialized successfully. Available formats:', Object.keys(formats).length);
            });

        } catch (error) {
            Log.error('Failed to initialize FFmpeg:', error);
            throw error;
        }
    }

    transcodeVideo(inputPath, outputPath, options = {}) {
        return new Promise((resolve, reject) => {
            // 保存 this 的引用
            const self = this;

            const command = ffmpeg(inputPath);

            const defaultOptions = {
                format: 'mp4',
                videoBitrate: '1000k',
                audioBitrate: '128k',
                frameRate: 30,
                width: 1280,
                height: 720
            };

            const finalOptions = { ...defaultOptions, ...options };

            command
                .format(finalOptions.format)
                .videoBitrate(finalOptions.videoBitrate)
                .audioBitrate(finalOptions.audioBitrate)
                .fps(finalOptions.frameRate)
                .size(`${finalOptions.width}x${finalOptions.height}`);

            command
                .on('start', (commandLine) => {
                    Log.info('Transcoding started:', commandLine);
                })
                .on('progress', function(progress) {
                    Log.info('Processing:', progress.percent, '% done');
                    // 使用保存的 self 引用来访问 ctx
                    try {
                        if (self.ctx && self.ctx.app && self.ctx.app.systemService) {
                            self.ctx.app.systemService.sendToWebContent('transcode-progress', {
                                percent: progress.percent,
                                time: progress.timemark
                            });
                        }
                    } catch (err) {
                        Log.error('Error sending progress:', err);
                    }
                })
                .on('end', () => {
                    Log.info('Transcoding completed');
                    resolve({
                        success: true,
                        outputPath: outputPath
                    });
                })
                .on('error', (err) => {
                    Log.error('Transcoding error:', err);
                    reject(err);
                });

            command.save(outputPath);
        });
    }

    /**
     * 获取视频信息
     * @param {string} filePath - 视频文件路径
     * @returns {Promise<Object>} 视频信息
     */
    getVideoInfo(filePath) {
        return new Promise((resolve, reject) => {
            ffmpeg.ffprobe(filePath, (err, metadata) => {
                if (err) {
                    Log.error('FFprobe error:', err);
                    reject(err);
                    return;
                }

                const videoStream = metadata.streams.find(stream => stream.codec_type === 'video');
                const audioStream = metadata.streams.find(stream => stream.codec_type === 'audio');

                const info = {
                    duration: metadata.format.duration,
                    size: metadata.format.size,
                    bitrate: metadata.format.bit_rate,
                    format: metadata.format.format_name,
                    video: videoStream ? {
                        codec: videoStream.codec_name,
                        width: videoStream.width,
                        height: videoStream.height,
                        frameRate: eval(videoStream.r_frame_rate),
                        bitrate: videoStream.bit_rate
                    } : null,
                    audio: audioStream ? {
                        codec: audioStream.codec_name,
                        channels: audioStream.channels,
                        sampleRate: audioStream.sample_rate,
                        bitrate: audioStream.bit_rate
                    } : null
                };

                resolve(info);
            });
        });
    }
}

VideoService.toString = () => '[class VideoService]';
module.exports = VideoService;
 => '[class VideoController]';
module.exports = VideoController;
```
