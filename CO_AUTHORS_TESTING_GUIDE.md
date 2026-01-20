# Co-Authors Feature Testing Guide

## Implementation Overview

This document describes the BASIC version of the co-authors feature that has been implemented.

## Features Implemented

### 1. Backend Changes
- **Article Entity**: Added `coAuthors` many-to-many relationship with User entity
- **Database Migration**: Created migration for `article_co_authors` junction table
- **Article Service**: 
  - `create()` method now accepts and stores co-authors from email addresses
  - `update()` method checks if user is author OR co-author before allowing edits
  - `findOne()` populates coAuthors when fetching articles
- **DTOs**: CreateArticleDto includes optional `coAuthors` field (string array of emails)

### 2. Frontend Changes
- **API Types**: Article interface includes `coAuthors?: Profile[]`
- **Article Edit Form**: Added "Co-Authors" input field for comma-separated email addresses
- **Data Transformation**: Effects transform comma-separated emails to array before API submission
- **Article Display**: Co-authors shown below article author with profile links
- **Edit Permissions**: Users can edit articles where they are author OR co-author

## How It Works (BASIC Version)

1. **Creating an Article with Co-Authors**:
   - User enters article details in the editor
   - In the "Co-Authors" field, user enters email addresses separated by commas
   - Example: `john@example.com, jane@example.com`
   - Upon submission, the emails are transformed into an array and sent to the backend
   - Backend looks up users by email and adds them as co-authors

2. **Editing an Article**:
   - Original author can always edit their articles
   - Co-authors can also edit the article
   - When multiple users edit simultaneously, **last save wins** (no locking in BASIC version)

3. **Viewing an Article**:
   - Article page shows the original author
   - Co-authors are displayed below the author information
   - Clicking on a co-author navigates to their profile

## Testing Instructions

### Prerequisites
1. Database running (MySQL)
2. Database migrated with `npx mikro-orm migration:up`
3. Database seeded with test users
4. Backend running on port 3000
5. Frontend running on port 4200

### Test Scenario 1: Create Article with Co-Author

**Given**: You are logged in as user "Zolly" (jcosten0@purevolume.com)

**Steps**:
1. Navigate to "New Article" page
2. Fill in article details:
   - Title: "Test Article with Co-Authors"
   - Description: "Testing co-authors feature"
   - Body: "This article has multiple authors"
   - Tags: "test"
   - Co-Authors: Enter email of "John" user (find in database/seeded users)
3. Click "Publish Article"

**Expected Result**:
- Article is created successfully
- Redirected to article view page
- Co-author "John" is displayed below the article author
- **[SCREENSHOT NEEDED]**: Capture the article view showing co-author

### Test Scenario 2: Co-Author Can Edit Article

**Given**: Article created by Zolly with John as co-author

**Steps**:
1. Log out from Zolly account
2. Log in as user "John"
3. Navigate to the article created in Test 1
4. Observe the "Edit Article" button is visible
5. Click "Edit Article"
6. Modify the article (change title or body)
7. Click "Publish Article"

**Expected Result**:
- John can see the Edit button (because he's a co-author)
- John can successfully edit the article
- Changes are saved
- **[SCREENSHOT NEEDED]**: Capture the article edit page showing John can edit

### Test Scenario 3: Concurrent Editing (Last Save Wins)

**Given**: Article created by Zolly with John as co-author

**Steps**:
1. Open article in browser window 1 (logged in as Zolly)
2. Click "Edit Article" in window 1
3. Open article in incognito browser window 2 (logged in as John)
4. Click "Edit Article" in window 2
5. In window 1 (Zolly): Change title to "Version by Zolly"
6. In window 2 (John): Change title to "Version by John"
7. In window 1 (Zolly): Click "Publish Article" - wait for save
8. In window 2 (John): Click "Publish Article" immediately after

**Expected Result** (BASIC Version):
- Both users can edit simultaneously
- No locking mechanism
- John's save overwrites Zolly's save (last save wins)
- Final article title is "Version by John"
- **[SCREENSHOT NEEDED]**: Capture showing both users could edit

## Database Schema

### New Table: `article_co_authors`
```sql
CREATE TABLE `article_co_authors` (
  `article_id` int unsigned NOT NULL,
  `user_id` int unsigned NOT NULL,
  PRIMARY KEY (`article_id`, `user_id`),
  KEY `article_co_authors_article_id_index` (`article_id`),
  KEY `article_co_authors_user_id_index` (`user_id`),
  CONSTRAINT `article_co_authors_article_id_foreign` 
    FOREIGN KEY (`article_id`) REFERENCES `article` (`id`) 
    ON UPDATE CASCADE ON DELETE CASCADE,
  CONSTRAINT `article_co_authors_user_id_foreign` 
    FOREIGN KEY (`user_id`) REFERENCES `user` (`id`) 
    ON UPDATE CASCADE ON DELETE CASCADE
);
```

## API Examples

### Create Article with Co-Authors
```bash
POST /api/articles
{
  "article": {
    "title": "Test Article",
    "description": "Test description",
    "body": "Test body",
    "tagList": ["test"],
    "coAuthors": ["john@example.com", "jane@example.com"]
  }
}
```

### Response includes co-authors
```json
{
  "article": {
    "slug": "test-article-xyz",
    "title": "Test Article",
    "description": "Test description",
    "body": "Test body",
    "tagList": ["test"],
    "coAuthors": [
      {
        "username": "john",
        "bio": "...",
        "image": "...",
        "following": false
      }
    ],
    "author": {
      "username": "zolly",
      "bio": "...",
      "image": "...",
      "following": false
    },
    "createdAt": "2026-01-20T18:00:00.000Z",
    "updatedAt": "2026-01-20T18:00:00.000Z",
    "favorited": false,
    "favoritesCount": 0
  }
}
```

## Known Limitations (BASIC Version)

1. **No Article Locking**: Multiple users can edit simultaneously, last save wins
2. **No Edit Conflict Detection**: Users aren't notified when someone else is editing
3. **No Lock Timeout**: N/A for BASIC version
4. **No Real-time Notifications**: Users don't see who else is editing
5. **Email-based Only**: Co-authors must be added by email, not by selecting from a list

## Future Enhancements (ADVANCED Version - Not Implemented)

1. Multi-select dropdown for adding co-authors from existing users
2. Article locking when a user starts editing
3. Lock timeout after 5 minutes of inactivity
4. Visual indication of who is currently editing
5. Error messages when trying to edit a locked article
6. Lock recovery on connection loss
7. Force unlock feature for original author

## Troubleshooting

### Co-author not added
- **Issue**: Email doesn't match any user in database
- **Solution**: Verify email exists in users table

### Can't edit article as co-author
- **Issue**: Database not migrated or permission check failing
- **Solution**: Run migrations and verify coAuthors are populated in article object

### Co-authors not displaying
- **Issue**: Frontend not populating coAuthors or backend not including in response
- **Solution**: Check network tab for API response, verify `populate: ['coAuthors']` in backend
