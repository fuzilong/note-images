## 整站加速，资源分离存OSS+CDN方案

1. 首页 Hero 图、产品图、新闻封面先迁 OSS + CDN  
2. 图片生成缩略图和大图  

3. 上传功能改为直接存 OSS  

4. 最后再把 JS/CSS assets 放 CDN



## 优势

- **ECS 只有一台源站**：用户不管在泰国、越南、菲律宾、印尼，静态资源都要回你的新加坡服务器取。（ECS只存储程序和日志）
- **OSS + CDN 有边缘缓存**：图片、JS、CSS 会被缓存到更靠近用户的 CDN 节点，命中缓存后不再打到你的 ECS。
- **减轻服务器压力**：你的 4G 内存、40G 磁盘其实够跑业务，但不适合长期承担大量图片传输。
- **缓存策略更好做**：图片、带 hash 的 JS/CSS 可以设置长缓存，页面 HTML/API 继续走 ECS。



## 前期必备

### OSS服务和配置

1. 开通OSS服务
2. 创建OSS Bucket,如：billiardvibe-assets，主要配置
   1. Bucket name
   2. 存储区域  OSS_REGION
   3. 其它按需或默认



### CDN配置

1. 开通CDN服务
2. 域名管理->添加域名
3. 基本配置
   1. 选择加速区域，域名无国内备案：全球（不包含中国国内）；有国内备案：全球
   2. 加速域名，输入站点cdn域名（域名服务上配置static.yourdomain.com），首次需要验证
      1. ![image-20260524143915272](https://raw.githubusercontent.com/fuzilong/images/main/note-img/image-20260524143915272.png)
      2. 阿里云给出CDN CNAME
   3. 业务类型选：图片/小文件/静态资源加速
   4. 源站类型选：OSS Domain
   5. 源站选择为项目创建的 OSS Bucket
4. 域名 DNS 里加一条 CNAME：static 记录
   1. static.billiardvibe.com ；类型：CNAME ；记录值：xxx.w.kunluncan.com（阿里云给的CDN CNAME）
5. **HTTPS**
   - 给 static.你的域名.com 配 SSL 证书
     - 阿里云申请SSL个人证书，绑定域名
     - 手动DNS验证
     - ![image-20260524145729851](https://raw.githubusercontent.com/fuzilong/images/main/note-img/image-20260524145729851.png)
   - 开启 HTTPS
   - 建议开启 HTTP 自动跳 HTTPS
6. **缓存规则**
   - 图片：30 天或更久
   - JS/CSS assets：1 年，前提是文件名带 hash
   - HTML 不放 CDN 或设置短缓存



## 图片存量迁移

1. 服务器批量导出&阿里云服务台批量导入

   1. 测试url

2. 代码和数据库批量更换url

   1. 数据库批量脚本

   2. ```sql
      -- Replace the legacy image CDN URLs with the new OSS + CDN URLs.
      -- Run this on local first, verify the site, then run it on production.
      
      SET @old_base = 'https://d2xsxph8kpxj0f.cloudfront.net/310519663413758317/Xy2BpgEYHKHrNFgjP93yWu';
      SET @new_base = 'https://static.billiardvibe.com/images';
      
      START TRANSACTION;
      
      UPDATE categories
      SET image = REPLACE(image, @old_base, @new_base)
      WHERE image LIKE CONCAT(@old_base, '/%');
      
      UPDATE news
      SET coverImage = REPLACE(coverImage, @old_base, @new_base)
      WHERE coverImage LIKE CONCAT(@old_base, '/%');
      
      UPDATE products
      SET thumbnailImage = REPLACE(thumbnailImage, @old_base, @new_base)
      WHERE thumbnailImage LIKE CONCAT(@old_base, '/%');
      
      UPDATE products
      SET images = REPLACE(CAST(images AS CHAR), @old_base, @new_base)
      WHERE CAST(images AS CHAR) LIKE CONCAT('%', @old_base, '/%');
      
      UPDATE siteConfig
      SET value = REPLACE(value, @old_base, @new_base)
      WHERE value LIKE CONCAT('%', @old_base, '/%');
      
      COMMIT;
      
      -- Verification: all counts should be 0 after migration.
      SELECT 'categories.image' AS field, COUNT(*) AS remaining
      FROM categories
      WHERE image LIKE CONCAT(@old_base, '/%')
      UNION ALL
      SELECT 'news.coverImage', COUNT(*)
      FROM news
      WHERE coverImage LIKE CONCAT(@old_base, '/%')
      UNION ALL
      SELECT 'products.thumbnailImage', COUNT(*)
      FROM products
      WHERE thumbnailImage LIKE CONCAT(@old_base, '/%')
      UNION ALL
      SELECT 'products.images', COUNT(*)
      FROM products
      WHERE CAST(images AS CHAR) LIKE CONCAT('%', @old_base, '/%')
      UNION ALL
      SELECT 'siteConfig.value', COUNT(*)
      FROM siteConfig
      WHERE value LIKE CONCAT('%', @old_base, '/%');
      ```

   3. 代码批量查找，批量替换

3. 设置Bucket公共读

4. 站点测试

### 图片尺寸参数式配置

 CDN 参数策略设置成：

![image-20260524151040997](https://raw.githubusercontent.com/fuzilong/images/main/note-img/image-20260524151040997.png)

```
保留指定参数：x-oss-process
```

保存配置成功后。

1. 新增一个图片工具函数，比如 getImageUrl(url, "thumb" | "card" | "detail" | "hero")
2. 产品列表、首页产品图、新闻封面用 thumb/card
3. 产品详情页主图用 detail
4. 首页大图用 hero
5. 数据库继续保持原图 URL，不新增字段



## Upload开发

到阿里云RAM访问控制

1. 创建一个用户，和项目名关联，如billiard-oss-uploader

2. 配置一个访问策略，或使用Aliyun的系统策略AliyunOSSFullAccess

3. 给用户授权，成功后生成AccessId和AccesKey

4. 配置项目OSS常量参数

   1. ```dockerfile
      OSS_REGION=oss-ap-southeast-1
      OSS_BUCKET=billiardvibe-assets
      OSS_ACCESS_KEY_ID=xxxxxxxx
      OSS_ACCESS_KEY_SECRET=xxxxxxxxxx
      OSS_UPLOAD_DIR=images
      OSS_PUBLIC_BASE_URL=https://static.billiardvibe.com
      ```

5. 引入对应开发语言的Ali OSS sdk依赖包

6. Codex完成上传功能代码开发