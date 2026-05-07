## What this guide is not

This guide does not describe removing CocoaPods from a React Native app entirely.

As of React Native 0.85.x, React Native itself and most native community modules still rely on CocoaPods for iOS integration.

This document focuses on migrating individual native Apple SDKs from CocoaPods to Swift Package Manager while continuing to use CocoaPods for the broader React Native ecosystem.

## Cursor Agent skill

A **Cursor Agent skill** (general Pod → SPM migration and troubleshooting for React Native iOS, any library) lives in [`.cursor/skills/rn-ios-cocoapods-spm-migration/SKILL.md`](.cursor/skills/rn-ios-cocoapods-spm-migration/SKILL.md). The long-form guide with **worked examples** (Firebase, sample stack, Kingfisher) is in [`docs/ios-cocoapods-to-spm-migration.md`](docs/ios-cocoapods-to-spm-migration.md).

## Visual architecture diagram

JavaScript Layer (npm)
│
├── @react-native-firebase/app
├── react-native-screens
└── react-native-safe-area-context
│
▼
Native iOS Layer
│
├── CocoaPods
│ ├── React-Core
│ ├── RNScreens
│ └── FirebaseCore
│
└── Swift Package Manager
└── Kingfisher
