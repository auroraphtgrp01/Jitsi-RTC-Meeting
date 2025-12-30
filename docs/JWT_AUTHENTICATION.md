# Jitsi Meet JWT Authentication Guide

## Tổng quan

Jitsi Meet sử dụng JWT (JSON Web Token) để xác thực người dùng và phân quyền moderator/participant trong phòng họp.

## Cấu hình

### Biến môi trường (.env)

```bash
# Bật xác thực
ENABLE_AUTH=1
AUTH_TYPE=jwt

# JWT Config
JWT_APP_ID=5F9D4                                    # Application ID (tùy chọn)
JWT_APP_SECRET=7F53ECFCC549E743D4D360F4684FE3B1     # Secret key để ký token (giữ bí mật!)
JWT_ALLOW_EMPTY=0                                   # Không cho phép token rỗng

# Tắt auto-owner để phân quyền từ JWT hoạt động
ENABLE_AUTO_OWNER=false
ENABLE_MODERATOR_CHECKS=true

# Kích hoạt module phân quyền
XMPP_MUC_MODULES=token_moderation
```

## Cấu trúc JWT Token

### Header
```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

### Payload

```json
{
  "aud": "5F9D4",                          // Audience - phải khớp với JWT_APP_ID
  "iss": "5F9D4",                          // Issuer - phải khớp với JWT_APP_ID
  "sub": "meeting.hoclientuc.vn",          // Subject - domain của Jitsi
  "room": "myroom",                        // Tên phòng (hoặc "*" cho tất cả phòng)
  "iat": 1767069488,                       // Issued At - thời điểm tạo token (Unix timestamp)
  "nbf": 1767069428,                       // Not Before - token có hiệu lực từ (Unix timestamp)
  "exp": 1767155888,                       // Expiration - token hết hạn (Unix timestamp)
  "context": {
    "user": {
      "id": "user123",                     // ID người dùng (tùy chọn)
      "name": "Nguyễn Văn A",              // Tên hiển thị
      "email": "user@example.com",         // Email (tùy chọn)
      "avatar": "https://...",             // URL ảnh đại diện (tùy chọn)
      "moderator": true                    // ⭐ QUAN TRỌNG: true = moderator, false = participant
    }
  }
}
```

### Giải thích các trường quan trọng

| Trường | Bắt buộc | Mô tả |
|--------|----------|-------|
| `aud` | ✅ | Audience - phải khớp với `JWT_APP_ID` trong .env |
| `iss` | ✅ | Issuer - phải khớp với `JWT_APP_ID` trong .env |
| `sub` | ✅ | Subject - domain của Jitsi Meet |
| `room` | ✅ | Tên phòng hoặc `*` cho tất cả phòng |
| `exp` | ✅ | Thời điểm hết hạn (Unix timestamp) |
| `context.user.name` | ✅ | Tên hiển thị của người dùng |
| `context.user.moderator` | ✅ | `true` = moderator, `false` = participant |

## Ví dụ Generate Token

### Python

```python
import jwt
import time

# Cấu hình từ .env
APP_ID = "5F9D4"
APP_SECRET = "7F53ECFCC549E743D4D360F4684FE3B1"
DOMAIN = "meeting.hoclientuc.vn"

def generate_token(user_id, user_name, user_email, room, is_moderator=False, expires_hours=24):
    """
    Generate JWT token cho Jitsi Meet
    
    Args:
        user_id: ID người dùng
        user_name: Tên hiển thị
        user_email: Email
        room: Tên phòng (hoặc "*" cho tất cả phòng)
        is_moderator: True nếu là moderator, False nếu là participant
        expires_hours: Số giờ token có hiệu lực
    
    Returns:
        JWT token string
    """
    now = int(time.time())
    
    payload = {
        "aud": APP_ID,
        "iss": APP_ID,
        "sub": DOMAIN,
        "room": room,
        "iat": now,
        "nbf": now - 60,  # Cho phép lệch 1 phút
        "exp": now + (expires_hours * 3600),
        "context": {
            "user": {
                "id": user_id,
                "name": user_name,
                "email": user_email,
                "moderator": is_moderator
            }
        }
    }
    
    return jwt.encode(payload, APP_SECRET, algorithm="HS256")


# Ví dụ sử dụng
if __name__ == "__main__":
    # Token cho MODERATOR
    mod_token = generate_token(
        user_id="admin01",
        user_name="Admin User",
        user_email="admin@example.com",
        room="myroom",
        is_moderator=True
    )
    print(f"Moderator URL: https://{DOMAIN}/myroom?jwt={mod_token}")
    
    # Token cho PARTICIPANT
    member_token = generate_token(
        user_id="user01",
        user_name="Normal User",
        user_email="user@example.com",
        room="myroom",
        is_moderator=False
    )
    print(f"Participant URL: https://{DOMAIN}/myroom?jwt={member_token}")
```

### Node.js

```javascript
const jwt = require('jsonwebtoken');

// Cấu hình từ .env
const APP_ID = "5F9D4";
const APP_SECRET = "7F53ECFCC549E743D4D360F4684FE3B1";
const DOMAIN = "meeting.hoclientuc.vn";

/**
 * Generate JWT token cho Jitsi Meet
 * @param {string} userId - ID người dùng
 * @param {string} userName - Tên hiển thị
 * @param {string} userEmail - Email
 * @param {string} room - Tên phòng (hoặc "*" cho tất cả phòng)
 * @param {boolean} isModerator - true nếu là moderator
 * @param {number} expiresHours - Số giờ token có hiệu lực
 * @returns {string} JWT token
 */
function generateToken(userId, userName, userEmail, room, isModerator = false, expiresHours = 24) {
    const now = Math.floor(Date.now() / 1000);
    
    const payload = {
        aud: APP_ID,
        iss: APP_ID,
        sub: DOMAIN,
        room: room,
        iat: now,
        nbf: now - 60,
        exp: now + (expiresHours * 3600),
        context: {
            user: {
                id: userId,
                name: userName,
                email: userEmail,
                moderator: isModerator
            }
        }
    };
    
    return jwt.sign(payload, APP_SECRET, { algorithm: 'HS256' });
}

// Ví dụ sử dụng
const modToken = generateToken("admin01", "Admin User", "admin@example.com", "myroom", true);
console.log(`Moderator URL: https://${DOMAIN}/myroom?jwt=${modToken}`);

const memberToken = generateToken("user01", "Normal User", "user@example.com", "myroom", false);
console.log(`Participant URL: https://${DOMAIN}/myroom?jwt=${memberToken}`);
```

### PHP

```php
<?php
require 'vendor/autoload.php';
use Firebase\JWT\JWT;

// Cấu hình từ .env
$APP_ID = "5F9D4";
$APP_SECRET = "7F53ECFCC549E743D4D360F4684FE3B1";
$DOMAIN = "meeting.hoclientuc.vn";

/**
 * Generate JWT token cho Jitsi Meet
 */
function generateToken($userId, $userName, $userEmail, $room, $isModerator = false, $expiresHours = 24) {
    global $APP_ID, $APP_SECRET, $DOMAIN;
    
    $now = time();
    
    $payload = [
        "aud" => $APP_ID,
        "iss" => $APP_ID,
        "sub" => $DOMAIN,
        "room" => $room,
        "iat" => $now,
        "nbf" => $now - 60,
        "exp" => $now + ($expiresHours * 3600),
        "context" => [
            "user" => [
                "id" => $userId,
                "name" => $userName,
                "email" => $userEmail,
                "moderator" => $isModerator
            ]
        ]
    ];
    
    return JWT::encode($payload, $APP_SECRET, 'HS256');
}

// Ví dụ sử dụng
$modToken = generateToken("admin01", "Admin User", "admin@example.com", "myroom", true);
echo "Moderator URL: https://{$DOMAIN}/myroom?jwt={$modToken}\n";

$memberToken = generateToken("user01", "Normal User", "user@example.com", "myroom", false);
echo "Participant URL: https://{$DOMAIN}/myroom?jwt={$memberToken}\n";
```

## Cách sử dụng Token

### URL Format

```
https://{DOMAIN}/{ROOM_NAME}?jwt={TOKEN}
```

Ví dụ:
```
https://meeting.hoclientuc.vn/myroom?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

## Phân quyền

### Moderator (`"moderator": true`)
- ✅ Kick người khác khỏi phòng
- ✅ Mute/Unmute người khác
- ✅ Kết thúc cuộc họp cho tất cả
- ✅ Bật/tắt lobby
- ✅ Cấp quyền moderator cho người khác
- ✅ Quản lý breakout rooms

### Participant (`"moderator": false`)
- ❌ Không thể kick người khác
- ❌ Không thể mute người khác
- ❌ Không thể kết thúc cuộc họp
- ✅ Chỉ có thể mute/unmute bản thân
- ✅ Bật/tắt camera bản thân
- ✅ Chat, share screen (nếu được phép)

## Lưu ý bảo mật

⚠️ **QUAN TRỌNG:**

1. **Giữ bí mật `JWT_APP_SECRET`** - Không được lộ secret key ra client-side code
2. **Tạo token ở server-side** - Luôn generate token ở backend, không bao giờ ở frontend
3. **Đặt thời hạn hợp lý** - Token nên có thời hạn ngắn (vài giờ đến 1 ngày)
4. **Xác thực user trước khi tạo token** - Đảm bảo user đã đăng nhập hệ thống của bạn
5. **Validate room name** - Kiểm tra user có quyền vào room đó không

## Troubleshooting

### Token không hoạt động

1. Kiểm tra `aud` và `iss` khớp với `JWT_APP_ID` trong .env
2. Kiểm tra token chưa hết hạn (`exp` > thời gian hiện tại)
3. Kiểm tra `sub` khớp với domain Jitsi
4. Restart containers sau khi thay đổi cấu hình

### Tất cả user đều là moderator

1. Đảm bảo `ENABLE_AUTO_OWNER=false` trong .env
2. Đảm bảo `XMPP_MUC_MODULES=token_moderation` trong .env
3. Kiểm tra file `mod_token_moderation.lua` có trong `/prosody-plugins-custom/`
4. Restart containers: `docker compose down && docker compose up -d`

### Xem logs

```bash
# Xem log Prosody
docker logs for-sns-prosody-1 2>&1 | grep -i "token\|moderation\|affiliation"

# Xem log Jicofo
docker logs for-sns-jicofo-1 2>&1 | grep -i "auth\|owner\|moderator"
```

