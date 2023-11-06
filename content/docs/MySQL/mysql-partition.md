---
title: "MySQL 分区表"
date: 2018-02-28T23:41:32+08:00
draft: true
tags: [mysql, ]
---

## 概述

MySQL 目前只支持水平分区，不支持垂直分区。

MySQL 的分区是局部分区，即数据和索引都放在分区中。另外还有全局分区，数据放在各个分区中，但索引放在同一个对象中，MySQL 目前还不支持全局分区。

## 分区类型

### Range 分区

### List 分区

### Hash 分区

### Key 分区

### Columns 分区

## 子分区

## 分区中的NULL值
