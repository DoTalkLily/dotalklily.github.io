---
layout: post
title:  ios推送功能添加以及模拟后端自测
date:   2017-01-04 08:43:59
author: Lily
categories: ios
tags:
- push notification
- apns
- node-apn
---
  1.apns简介：
     参见：[apns原理](https://developer.apple.com/library/content/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/APNSOverview.html#//apple_ref/doc/uid/TP40008194-CH8-SW1)
  2. 配置证书
     参见: [ios官网](https://developer.apple.com/library/content/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/HandlingRemoteNotifications.html#//apple_ref/doc/uid/TP40008194-CH6-SW1) 和 [这里](https://github.com/node-apn/node-apn/wiki/Preparing-Certificates)
     将上面生成的两个pem文件 key.pem 和 cert.pem 以及app id 给后端同学即可。
  3. ios接受处理push逻辑

  TKMPushNotificationCenter.h


 ```ios

 //  TKMPushNotificationCenter.h
 //
 //  Created by 01 on 16/12/17.
 //  Copyright © 2016年 Facebook. All rights reserved.
 //

 #import <Foundation/Foundation.h>
 #import <UIKit/UIKit.h>
 /**
  *  推送相关
  */

 @interface TKMPushNotificationCenter : NSObject
 /**
  *  Device Token
  */
 @property (nonatomic, copy, readonly) NSString *deviceTokenString;


 /**
  *  推送单例
  */
 + (instancetype)sharedPushCenter;

 /**
 *  是否开启系统的APNs
 */
 + (BOOL)APNsEnabled;

 /**
  *  是否从APNs的通知启动APP
  */
 + (BOOL)isLaunchFromAPNs;

 /**
  *  注册远程通知<向用户请求，获取推送通知权限>
  */
 + (void)registerRemoteNotificationAfterDelay:(NSTimeInterval)delay;

 /**
  *  iOS10推送，APP enter foreground
  */
 + (void)applicationWillEnterForegroundForIOS10Push:(UIApplication *)application;

 /**
  *  当APP处于terminate状态，通过通知启动APP，launchOptions中包含远程通知信息
  */
 + (void)application:(UIApplication *)application didReceiveRemoteNotificationFromLaunchingWithOptions:(NSDictionary *)launchOptions;

 /**
  *  注册用户通知设置
  */
 + (void)application:(UIApplication *)application didRegisterUserNotificationSettings:(UIUserNotificationSettings *)notificationSettings;

 /**
  *  注册远程通知成功
  */
 + (void)application:(UIApplication *)application didRegisterForRemoteNotificationsWithDeviceToken:(NSData *)deviceToken;

 /**
  *  注册远程通知失败
  */
 + (void)application:(UIApplication *)application didFailToRegisterForRemoteNotificationsWithError:(NSError *)error;

 /**
  *  当APP处理background状态时，点击收到的远程通知会调用该方法
  *  当APP处理foreground状态时，会直接调用该方法
  */
 + (void)application:(UIApplication *)application didReceiveRemoteNotification:(NSDictionary *)userInfo;

 /**
  *  收到本地通知
  */
 + (void)application:(UIApplication *)application didReceiveLocalNotification:(UILocalNotification *)notification;

 @end
 ```

  TKMPushNotificationCenter.m

 ```ios
 //
 //  TKMPushNotificationCenter.m
 //  kuaima
 //
 //  Created by 01 on 16/12/17.
 //  Copyright © 2016年 Facebook. All rights reserved.
 //

 #import <UserNotifications/UserNotifications.h>
 #import "TKMPushNotificationCenter.h"
 #import <TTInfoHelper.h>
 #import <NSObject+TTAdditions.h>

 /**
 注： ttinfohelper是ios组封装的一个查询设备操作系统版本的工具类，此处只用到判断当前系统的方法
 */


 NSString * const TKMRemoteNotificationDeviceTokenKey = @"toutiao.kuaima.push_notification.device_token";

 /**
  *  推送通知block
  */
 typedef void(^TEUPushNotificationCompletionBlock) (void);

 @interface TKMPushNotificationCenter () <Singleton,UNUserNotificationCenterDelegate>
 /**
  *  标记是否从通知栏启动APP
  */
 @property (nonatomic, assign) BOOL launchFromAPNs;

 @property (nonatomic, strong) NSDictionary *launchOptions;

 @property (nonatomic,   copy) TEUPushNotificationCompletionBlock completionBlock;
 /**
  *  Push Notification device token
  */
 @property (nonatomic, copy, readwrite) NSString *deviceTokenString;
 @end
 @implementation TKMPushNotificationCenter
 @synthesize deviceTokenString = _deviceTokenString;

 + (instancetype)sharedPushCenter
 {
   return [self sharedInstance_tt];
 }

 - (instancetype)init
 {
   if ((self = [super init])) {
     _launchFromAPNs = NO;
       }
   return self;
 }

 - (void)dealloc
 {
   _launchFromAPNs = NO;
   _launchOptions  = nil;
   [[NSNotificationCenter defaultCenter] removeObserver:self];
 }

 - (void)handleDidEyeuLogin:(NSNotification *)notification
 {
   [self registerRemoteNotificationAfterDelay:2.f];
 }


 + (BOOL)isLaunchFromAPNs
 {
   return [[TKMPushNotificationCenter sharedPushCenter] launchFromAPNs];
 }

 #pragma mark - handle remote notification

 + (void)application:(UIApplication *)application didReceiveRemoteNotificationFromLaunchingWithOptions:(NSDictionary *)launchOptions
 {
   [[self sharedPushCenter] application:application didReceiveRemoteNotificationFromLaunchingWithOptions:launchOptions];
 }

 - (void)application:(UIApplication *)application didReceiveRemoteNotificationFromLaunchingWithOptions:(NSDictionary *)launchOptions
 {
   // 注册远程通知
   [self registerRemoteNotificationAfterDelay:0.f];


   // 处理从通知栏点击通知启动App的情况【iOS10之后会调用[userNotificationCenter:didReceiveNotificationResponse]】
   BOOL isFromAPNs = [launchOptions objectForKey:UIApplicationLaunchOptionsRemoteNotificationKey] != nil;
   [TKMPushNotificationCenter sharedPushCenter].launchFromAPNs = YES;
   [TKMPushNotificationCenter sharedPushCenter].launchOptions = [launchOptions copy];

   if ([TTInfoHelper OSVersionNumber] < 10.0 && isFromAPNs) {
     // apns should be handled after mainViewController is load completely
     NSDictionary *payload = [launchOptions objectForKey:UIApplicationLaunchOptionsRemoteNotificationKey];
     [[TKMPushNotificationCenter sharedPushCenter] handleRemoteNotification:payload delay:1.f];
   }
 }

 + (void)application:(UIApplication *)application didRegisterUserNotificationSettings:(UIUserNotificationSettings *)notificationSettings
 {
   [[self sharedPushCenter] application:application didRegisterUserNotificationSettings:notificationSettings];
 }

 + (void)application:(UIApplication *)application didRegisterForRemoteNotificationsWithDeviceToken:(NSData *)deviceToken
 {
   [[self sharedPushCenter] application:application didRegisterForRemoteNotificationsWithDeviceToken:deviceToken];
 }

 + (void)application:(UIApplication *)application didFailToRegisterForRemoteNotificationsWithError:(NSError *)error
 {
   [[self sharedPushCenter] application:application didFailToRegisterForRemoteNotificationsWithError:error];
 }

 + (void)application:(UIApplication *)application didReceiveRemoteNotification:(NSDictionary *)userInfo
 {
   [[self sharedPushCenter] application:application didReceiveRemoteNotification:userInfo];
 }

 + (void)application:(UIApplication *)application didReceiveLocalNotification:(UILocalNotification *)notification
 {
   [[self sharedPushCenter] application:application didReceiveLocalNotification:notification];
 }

 - (void)application:(UIApplication *)application didRegisterUserNotificationSettings:(UIUserNotificationSettings *)notificationSettings
 {
   if (notificationSettings.types != UIUserNotificationTypeNone) {
     [application registerForRemoteNotifications];
   }
 }

 - (void)application:(UIApplication *)application didRegisterForRemoteNotificationsWithDeviceToken:(NSData *)deviceToken
 {
   self.deviceTokenString = [[[[deviceToken description]
                               stringByReplacingOccurrencesOfString: @"<" withString: @""]
                              stringByReplacingOccurrencesOfString: @">" withString: @""]
                             stringByReplacingOccurrencesOfString: @" " withString: @""];
   NSLog(@">>>>> Success: device token: %@", self.deviceTokenString);
 }

 - (void)application:(UIApplication *)application didFailToRegisterForRemoteNotificationsWithError:(NSError *)error
 {
   NSLog(@">>>>> Failed: registerRemoteNotificiation: %@", error);
 }

 - (void)application:(UIApplication *)application didReceiveRemoteNotification:(NSDictionary *)userInfo
 {
   NSLog(@">>>>> Launch: receiveRemoteNotification: %@",userInfo);

   application.applicationIconBadgeNumber = [[userInfo objectForKey:@"badge"] integerValue];
   if ([[UIApplication sharedApplication] applicationState] != UIApplicationStateActive) {
     [self handleRemoteNotification:userInfo];
   } else {
     if ([[userInfo objectForKey:@"importance"] isEqualToString:@"important"]) {

     }
   }
 }

 #pragma mark - iOS10 UNNotificationCenterDelegate

 + (void)applicationWillEnterForegroundForIOS10Push:(UIApplication *)application
 {
   if ([TTInfoHelper OSVersionNumber] >= 10.0) {
     dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{

     });
   }
 }

 - (void)userNotificationCenter:(UNUserNotificationCenter *)center willPresentNotification:(UNNotification *)notification withCompletionHandler:(void (^)(UNNotificationPresentationOptions options))completionHandler
 {
   NSDictionary *payload __unused = notification.request.content.userInfo;
   UNNotificationRequest *request = notification.request; // 收到推送的请求
   UNNotificationContent *content = request.content; // 收到推送的消息内容
   NSNumber *badge __unused = content.badge;  // 推送消息的角标
   NSString *body  __unused= content.body;    // 推送消息体
   UNNotificationSound *sound __unused= content.sound;  // 推送消息的声音
   NSString *subtitle __unused = content.subtitle;  // 推送消息的副标题
   NSString *title __unused = content.title;  // 推送消息的标题

   if([request.trigger isKindOfClass:[UNPushNotificationTrigger class]]) {

   }
   else {
     // 判断为本地通知

   }

   // 需要执行这个方法，选择是否提醒用户，有Badge、Sound、Alert三种类型可以设置
   completionHandler(UNNotificationPresentationOptionBadge | UNNotificationPresentationOptionSound | UNNotificationPresentationOptionAlert);
 }

 - (void)userNotificationCenter:(UNUserNotificationCenter *)center didReceiveNotificationResponse:(UNNotificationResponse *)response withCompletionHandler:(void (^)())completionHandler
 {
   NSDictionary *payload = response.notification.request.content.userInfo;
   UNNotificationRequest *request = response.notification.request; // 收到推送的请求

   NSLog(@">>>>>iOS10: UNNotificationResponse: %@, payload: %@", response, payload);

   if ([request.trigger isKindOfClass:[UNPushNotificationTrigger class]]) {
     // 远程通知
     if ([self.launchOptions objectForKey:UIApplicationLaunchOptionsRemoteNotificationKey]) {
       self.launchOptions = nil;
       [self handleRemoteNotification:payload delay:1.f];
     } else {
       [self handleRemoteNotification:payload];
     }
   }


   if ([response.actionIdentifier isEqualToString:UNNotificationDefaultActionIdentifier]) {

   }
   else if ([response.actionIdentifier isEqualToString:UNNotificationDismissActionIdentifier]) {

   }
   else {

   }

   completionHandler();
 }


 #pragma mark - register remote notification
 /**
  *  注册远程通知<向用户请求，获取推送通知权限>
  */
 + (void)registerRemoteNotificationAfterDelay:(NSTimeInterval)delay
 {
   [[self sharedPushCenter] registerRemoteNotificationAfterDelay:delay];
 }

 - (void)registerRemoteNotificationAfterDelay:(NSTimeInterval)delay
 {
   void (^TEURegisterRemoteNotificationBlock)() = ^() {
     if ([TTInfoHelper OSVersionNumber] >= 10.0) {
       UNUserNotificationCenter *userNotificationCenter = [UNUserNotificationCenter currentNotificationCenter];
       userNotificationCenter.delegate = self;
       [userNotificationCenter requestAuthorizationWithOptions:UNAuthorizationOptionAlert | UNAuthorizationOptionBadge | UNAuthorizationOptionSound completionHandler:^(BOOL granted, NSError * _Nullable error) {
         if (granted) {
           NSLog(@"iOS10 > 注册通知成功");
           // initiate to get device token from APNs
           [[UIApplication sharedApplication] registerForRemoteNotifications];
         } else {
           NSLog(@"iOS10 > 注册通知失败");
         }
       }];
     }
     else if ([TTInfoHelper OSVersionNumber] >= 8.0) {
 #pragma clang diagnostic push
 #pragma clang diagnostic ignored "-Wdeprecated-declarations"
       UIUserNotificationSettings *settings = [UIUserNotificationSettings settingsForTypes:(UIRemoteNotificationTypeBadge | UIRemoteNotificationTypeSound | UIRemoteNotificationTypeAlert) categories:nil];
       [[UIApplication sharedApplication] registerUserNotificationSettings:settings];
 #pragma clang diagnostic pop
     }
     else {
 #pragma clang diagnostic push
 #pragma clang diagnostic ignored "-Wdeprecated-declarations"
       [[UIApplication sharedApplication] registerForRemoteNotificationTypes:(UIRemoteNotificationTypeBadge | UIRemoteNotificationTypeAlert | UIRemoteNotificationTypeSound)];
 #pragma clang diagnostic pop
     }
   };

   if(delay > 0) {
     dispatch_after(dispatch_time(DISPATCH_TIME_NOW, delay * NSEC_PER_SEC), dispatch_get_main_queue(), ^(void){
       TEURegisterRemoteNotificationBlock();
     });
   } else {
     TEURegisterRemoteNotificationBlock();
   }
 }

 #pragma mark - handle when receiving remote push


 - (void)handleRemoteNotification:(NSDictionary *)playload delay:(NSTimeInterval)delay
 {
   if (delay > 0) {
     dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(delay * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
       [self handleRemoteNotification:playload];
     });
   } else {
     [self handleRemoteNotification:playload];
   }
 }

 - (void)handleRemoteNotification:(NSDictionary *)playload
 {
   NSLog(@"handleRemoteNotification -> playload = %@", playload);

   [self.class clearAppBadgeAndNotificationCenter];
   //此处添加payload处理逻辑
 }

 + (void)clearAppBadgeAndNotificationCenter
 {
   // iOS只会在有badge的情况下清除通知中心的消息，因此需要先设置badge为1才能保证用户每次点击都能清楚通知中心的消息
   [[UIApplication sharedApplication] setApplicationIconBadgeNumber:1];
   [[UIApplication sharedApplication] setApplicationIconBadgeNumber:0];
   [[UIApplication sharedApplication] cancelAllLocalNotifications];
 }

 #pragma mark - setter/getter

 - (NSString *)deviceTokenString
 {
   if (_deviceTokenString) {
     return _deviceTokenString;
   }

   _deviceTokenString = [[NSUserDefaults standardUserDefaults] valueForKey:TKMRemoteNotificationDeviceTokenKey];
   return _deviceTokenString;
 }

 - (void)setDeviceTokenString:(NSString *)deviceTokenString
 {
   if (_deviceTokenString != deviceTokenString) {
     _deviceTokenString = deviceTokenString;

     [[NSUserDefaults standardUserDefaults] setObject:_deviceTokenString forKey:TKMRemoteNotificationDeviceTokenKey];
     [[NSUserDefaults standardUserDefaults] synchronize];
   }
 }

 @end

 ```

  4. 模拟后端发送push消息：
     所用组件库：https://github.com/node-apn/node-apn
 简易版推送源码如下：
 ```js
 "use strict";

 /**
  Send individualised notifications
  */

 const apn = require("apn");

 //内容随意定制，可以写多个想要测试的设备device token，device token从(void)application:(UIApplication *)application didRegisterForRemoteNotificationsWithDeviceToken:(NSData *)deviceToken回调函数中获得
 var users = [
   { content: "baby", "devices": ["2c3267a5f1e8d075ecc41d4cfa28b2df9b80a72d870f29d4c714cec0a0c59307"]}];


 var service = new apn.Provider({
   cert: "cert.pem", //路径根据文件位置改动
   key: "key.pem", //路径根据文件位置改动
 });

 users.forEach( function(user){

   var note = new apn.Notification({
       alert: users.content, //自己定制，可以写成根据不同user发不同内容
       payload: {
           "liveId": 5719,
       }
   });

   //app id 非常重要，要和应用的保持一致
   note.topic = "com.bytedance.kuaima";

   console.log('Sending:{} to {}',note.compile(),user.devices);

   service.send(note, user.devices).then( function(result) {
       console.log("sent:", result.sent.length);
       console.log("failed:", result.failed.length);
       console.log(result.failed);
   });
 });

 // For one-shot notification tasks you may wish to shutdown the connection
 // after everything is sent, but only call shutdown if you need your
 // application to terminate.
 service.shutdown();
 ```