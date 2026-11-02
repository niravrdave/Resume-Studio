# Auth Testing Playbook (CareerTax AI)

## JWT email/password
```
API=https://careertax-ai.preview.emergentagent.com/api
curl -c cookies.txt -X POST $API/auth/login -H "Content-Type: application/json" -d '{"email":"nirav.r.dave@gmail.com","password":"CareerTax@2026!"}'
curl -b cookies.txt $API/auth/me
```
- bcrypt hashes start with `$2b$`; indexes: users.email unique, login_attempts.identifier, email_tokens.expires_at (TTL).
- 5 failed logins for the same ip+email => 429 lockout 15 min.

## Emergent Google auth - create a test session
```
mongosh --eval "
use('test_database');
var userId='user_test' + Date.now(); var tok='test_session_' + Date.now();
db.users.insertOne({user_id:userId,email:'test.user.'+Date.now()+'@example.com',name:'Test User',role:'user',email_verified:true,auth_providers:['google'],created_at:new Date().toISOString()});
db.user_sessions.insertOne({user_id:userId,session_token:tok,expires_at:new Date(Date.now()+7*864e5),created_at:new Date().toISOString()});
print(tok);"
curl -H "Authorization: Bearer <tok>" $API/auth/me
```
Browser: add cookie `session_token` (domain careertax-ai.preview.emergentagent.com, path /, secure, httpOnly, SameSite=None).

## Cleanup
```
mongosh --eval "use('test_database'); db.users.deleteMany({email:/^(test\.user\.|test_)/i}); db.user_sessions.deleteMany({session_token:/test_session/});"
```
