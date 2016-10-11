---
title: ios麦克风权限申请
categories: ios
date: 2016-10-11 14:07:39
timestamp:
catAlias:
tags:
keywords:
description:
---
# ios麦克风权限申请

# 权限申请

```
- (BOOL)canRecord
{
    __block BOOL bCanRecord = YES;

    if ([[[UIDevice currentDevice] systemVersion] compare:@"7.0"] != NSOrderedAscending)
    {
        AVAudioSession *audioSession = [AVAudioSession sharedInstance];
        if ([audioSession respondsToSelector:@selector(requestRecordPermission:)]) {
            [audioSession performSelector:@selector(requestRecordPermission:) withObject:^(BOOL granted) {
                if (granted) {
                    bCanRecord = YES;
                } else {
                    bCanRecord = NO;
                }
            }];
        }
    }
    
    return bCanRecord;
}
```

# ios 10 麦克风访问崩溃

ios 10上除了申请麦克风权限之外，还需要在plist文件中添加一对kv，否则会出现崩溃。

如: 

`Privacy - Microphone Usage Description`: `xxx程序将要访问你的麦克风， 是否允许?`



