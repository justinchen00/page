# Billing Service

## 入库

1. 新建personprofile index

2. 有新人并且是名人时，对此index进行更新。 以celebrity_name为key。

   ```json
   {
       "celebrity_name", 
       "person_id",
       "group_id",
       "peer_id",
       "picture_path"，
       "timestamp",
   }
   ```

   /metadataproto.MetadataUploader.PushPicInfo  

3. 不是新人，但是是LIVE第一次出现时，更新personprofile里的 `peer_id` 列表

## 出库

人名搜索直接落在 personprofile,  命中的条件是：

1. 人名匹配
2. 用户有权限的`peer_id` 列表 和 人名后的 peer_id 列表有交集



如果通过peer_id 列表过滤有效率问题，可妥协地 用GroupID做过滤。















1. 