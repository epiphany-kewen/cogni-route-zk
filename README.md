# cogni-route-zk

-- KEYS[1]: expert_slot_key (e.g., "slot:expert_8848:20261120_1600")
-- KEYS[2]: user_credit_key (e.g., "credit:user_1024")
-- ARGV[1]: lock_owner_id, ARGV[2]: required_credit, ARGV[3]: ttl_ms

if redis.call("EXISTS", KEYS[1]) == 1 then
    return -1 -- 时间片已被锁定
end

local current_credit = tonumber(redis.call("GET", KEYS[2]) or "0")
if current_credit < tonumber(ARGV[2]) then
    return -2 -- 认知积分不足
end

redis.call("DECRBY", KEYS[2], ARGV[2])
redis.call("SET", KEYS[1], ARGV[1], "PX", ARGV[3])
return 1 -- 锁定成功，进入 15 分钟 Escrow 托管状态


from reclaim_protocol_sdk import verify_proof
import asyncpg

async def verify_and_route(zk_proof: dict, query_vec: list[float], db_pool: asyncpg.Pool):
    # 1. 链下密码学验证 TLS 流量零知识证明，锁定医院/律所/高校域名归属
    is_valid, extracted_ctx = verify_proof(
        zk_proof, 
        allowed_domains=["@pumch.cn", "@kwm.com", ".edu"]
    )
    if not is_valid:
        raise PermissionError("ZK-TLS Sybil check failed")

    # 2. pgvector 余弦相似度结合节点信誉权重重排序
    sql = """
        SELECT node_id, domain_tag,
               (1 - (expertise_embedding <=> $1::vector)) * 0.7 + (rep_score * 0.3) AS final_score
        FROM verified_expert_nodes
        WHERE status = 'IDLE' AND zk_verified = TRUE
        ORDER BY final_score DESC LIMIT 1;
    """
    async with db_pool.acquire() as conn:
        return await conn.fetchrow(sql, str(query_vec))
