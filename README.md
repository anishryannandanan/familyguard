# FamilyGuard - Parental Control & Monitoring System

## Overview
FamilyGuard is a comprehensive parental control and monitoring solution for Android devices, designed for parents/legal guardians to monitor their minor children's digital activities. The system consists of three main components:

1. **Child App** - Installed on the child's device for monitoring and control
2. **Parent App** - Installed on the parent's device for management and viewing
3. **Backend Server** - Cloud service for data synchronization and remote commands

## Key Features
- Social media monitoring (WhatsApp, Instagram, TikTok, Facebook, Snapchat)
- Keylogging via AccessibilityService
- Screen capture and recording via MediaProjection
- Camera and microphone access for remote monitoring
- Location tracking and geofencing
- App usage time and blocking
- Website filtering and content blocking
- Notification history logging
- Remote commands (lock, ring, wipe, message)
- Stealth mode operation
- Device Owner provisioning for uninstall prevention

## Legal & Ethical Considerations
⚠️ **IMPORTANT WARNING**: This software is designed for monitoring devices owned by parents/legal guardians for monitoring their minor children. Users must comply with all local laws and regulations. Keylogging, silent camera/mic access, and stealth mode may be illegal without proper consent in some jurisdictions.

## Project Structure
- `/docs/` - Documentation (PRD, architecture, guides)
- `/android-parent/` - Parent management app
- `/android-child/` - Child monitoring app
- `/backend/` - Server-side code and API
- `/configs/` - Configuration files
- `/tools/` - Development tools and scripts
- `/scripts/` - Build and deployment scripts

## Getting Started
See individual component README files for setup instructions.

## Compliance
- India IT Act 2000
- DPDP Act 2023 (India)
- GDPR considerations (if applicable)
- COPPA (Children's Online Privacy Protection Act) considerations

## License
This project is for educational and authorized parental use only. Commercial distribution requires proper legal review and compliance measures.