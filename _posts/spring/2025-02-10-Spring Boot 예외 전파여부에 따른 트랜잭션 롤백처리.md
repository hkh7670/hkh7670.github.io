---
title: Spring 예외 전파여부에 따른 트랜잭션 롤백처리
author: hkh7670
date: 2025-02-10 00:00:00 +0900
categories: [Spring]
tags: [Spring, Spring Boot, Transaction, DB]
---

## Case 1
```java
@Transactional
public void insertPopularMovies() { 
  PopularMoviePageResponse res = getPopularMovies(1);
  movieRepository.saveAll(
      res.results().stream()
          .map(item -> MovieEntity.of(item, 1))
          .toList()
  );
  PopularMoviePageResponse res2 = getPopularMovies(2);
  tmdb2Service.insertPopularMovies2(res2);
}
```
```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void insertPopularMovies2(PopularMoviePageResponse res) {
  movieRepository.saveAll(
      res.results().stream()
          .map(item -> MovieEntity.of(item, 2))
          .toList()
  );
  throw new RuntimeException();
}
```
- insertPopularMovies에서 insertPopularMovies2를 호출한 형태
- insertPopularMovies2에서 임의로 RuntimeException을 throw 했다.
- insertPopularMovies2에서 `REQUIRES_NEW`에 의해 트랜잭션이 분리되었어도 insertPopularMovies2의 RuntimeException()으로 인해 예외가 상위 메소드로 전파되어 insertPopularMovies의 내용도 롤백됨


## Case 2
```java
@Transactional
public void insertPopularMovies() {
  PopularMoviePageResponse res = getPopularMovies(1);
  movieRepository.saveAll(
      res.results().stream()
          .map(item -> MovieEntity.of(item, 1))
          .toList()
  );
  PopularMoviePageResponse res2 = getPopularMovies(2);
  try {
    tmdb2Service.insertPopularMovies2(res2);
  } catch (Exception e) {
    log.info(e.getMessage());
  }
}
```
```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void insertPopularMovies2(PopularMoviePageResponse res) {
  movieRepository.saveAll(
      res.results().stream()
          .map(item -> MovieEntity.of(item, 2))
          .toList()
  );
  throw new RuntimeException();
}
```
- insertPopularMovies에서 insertPopularMovies2를 호출한 형태
- 이번엔 insertPopularMovies2를 try catch로 묶어서 호출
- catch에서 log만 찍고 exception을 throw하지 않으므로 insertPopularMovies에서 작업한 내용은 롤백되지 않는다.

## Case 3
```java
@Transactional
public void insertPopularMovies() {
  PopularMoviePageResponse res = getPopularMovies(1);
  movieRepository.saveAll(
      res.results().stream()
          .map(item -> MovieEntity.of(item, 1))
          .toList()
  );
  PopularMoviePageResponse res2 = getPopularMovies(2);
  tmdb2Service.insertPopularMovies2(res2);
  throw new RuntimeException();
}
```
```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void insertPopularMovies2(PopularMoviePageResponse res) {
  movieRepository.saveAll(
      res.results().stream()
          .map(item -> MovieEntity.of(item, 2))
          .toList()
  );
}
```
- insertPopularMovies에서 insertPopularMovies2를 호출한 형태
- 이번엔 insertPopularMovies2 작업까지 마치고 insertPopularMovies의 마지막 부분에서 RuntimeException을 throw 했다.
- 이 경우, insertPopularMovies2는 새로운 트랜잭션으로 묶여서 commit 까지 완료되었으므로 롤백되지 않고
- insertPopularMovies에서 작업한 내용만 롤백된다.