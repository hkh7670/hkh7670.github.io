---
title: Jpa Batch Insert
author: hkh7670
date: 2024-03-27 00:00:00 +0900
categories: [test]
tags: [Spring Boot, DB, SQL]
---

# JPA Batch Insert

## Code

```java
public int saveAll(List<ImageInfo> imageInfoList) {
  int batchCount = 0;
  int totalSize = imageInfoList.size();
  if (totalSize <= BATCH_SIZE) {
    insert(imageInfoList, batchCount);
    return batchCount;
  }

  List<ImageInfo> subList = new ArrayList<>();
  for (int i = 0; i < totalSize; i++) {
    subList.add(imageInfoList.get(i));
    if ((i + 1) % BATCH_SIZE == 0) {
      batchCount = insert(subList, batchCount);
    }
  }
  return batchCount;
}

private int insert(List<ImageInfo> request, int batchCount) {
  jdbcTemplate.batchUpdate(
      "INSERT INTO IMAGE_INFO("
          + "IS_PRODUCT,"
          + "LINK,"
          + "ORIGINAL,"
          + "ORIGINAL_HEIGHT,"
          + "ORIGINAL_WIDTH,"
          + "POSITION,"
          + "RELATED_CONTENT_ID,"
          + "SERPAPI_RELATED_CONTENT_LINK,"
          + "SOURCE,"
          + "SOURCE_LOGO,"
          + "THUMBNAIL,"
          + "TITLE) VALUES(?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)",
      request, BATCH_SIZE, (ps, argument) -> {
        ps.setBoolean(1, argument.getIsProduct());
        ps.setString(2, argument.getLink());
        ps.setString(3, argument.getOriginal());
        ps.setInt(4, argument.getOriginalHeight());
        ps.setInt(5, argument.getOriginalWidth());
        ps.setLong(6, argument.getPosition());
        ps.setString(7, argument.getRelatedContentId());
        ps.setString(8, argument.getSerpapiRelatedContentLink());
        ps.setString(9, argument.getSource());
        ps.setString(10, argument.getSourceLogo());
        ps.setString(11, argument.getThumbnail());
        ps.setString(12, argument.getTitle());
      });
  request.clear();
  batchCount++;
  return batchCount;
}
```
