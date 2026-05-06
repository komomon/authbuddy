# Datasink Patterns — Authorization-Relevant Operations

Datasinks are operations that access, modify, or transmit resources — and thus require authorization. Each datasink MUST be checked for auth anchor binding.

---

## 1. Database Operations

### 1.1 Read Operations (db_read)

**Java:**
- `JdbcTemplate.query(...)`, `JdbcTemplate.queryForObject(...)`, `JdbcTemplate.queryForList(...)`
- `EntityManager.find(...)`, `EntityManager.createQuery(...).getResultList()`
- `JpaRepository.findById(...)`, `JpaRepository.findAll(...)`, `JpaRepository.findByXxx(...)` (Spring Data query methods)
- `MyBatis SqlSession.selectOne(...)`, `SqlSession.selectList(...)`
- `Hibernate Session.get(...)`, `Session.createQuery(...)`, `CriteriaBuilder`
- `@Query` annotated repository methods
- Raw `Statement.executeQuery(...)`, `PreparedStatement.executeQuery(...)`

**Python:**
- `cursor.execute("SELECT ...")`, `cursor.fetchone()`, `cursor.fetchall()`
- Django ORM: `Model.objects.get(...)`, `Model.objects.filter(...)`, `Model.objects.all()`
- SQLAlchemy: `session.query(...).filter(...).all()`, `session.execute(select(...))`

**Go:**
- `db.Query(...)`, `db.QueryRow(...)`, `db.Select(...)`
- GORM: `db.Find(...)`, `db.First(...)`, `db.Where(...).Find(...)`

**JavaScript/TypeScript:**
- `pool.query('SELECT ...')`, `client.query('SELECT ...')`
- Prisma: `prisma.model.findMany(...)`, `prisma.model.findUnique(...)`
- TypeORM: `repository.find(...)`, `repository.findOne(...)`
- Sequelize: `Model.findAll(...)`, `Model.findOne(...)`

### 1.2 Write Operations (db_write)

Same ORM/driver patterns but for:
- `INSERT`, `UPDATE`, `DELETE`, `MERGE`, `REPLACE`
- `repository.save(...)`, `repository.delete(...)`, `repository.saveAll(...)`
- `session.persist(...)`, `session.merge(...)`, `session.remove(...)`
- `db.Exec(...)`, `db.Create(...)`, `db.Save(...)`, `db.Delete(...)`
- Bulk operations: `batchInsert`, `bulkUpdate`, `bulkDelete`

---

## 2. File System Operations

### 2.1 File Read (file_read)

**Java:**
- `Files.readAllBytes(...)`, `Files.newInputStream(...)`, `Files.readString(...)`
- `FileInputStream(...)`, `FileReader(...)`, `BufferedReader(...)`
- `ClassPathResource.getInputStream()`, `ResourceUtils.getFile(...)`
- `new Scanner(new File(...))`

**Python:**
- `open(file_path, 'r')`, `Path(file_path).read_text()`, `Path(file_path).read_bytes()`
- `io.open(...)`, `codecs.open(...)`

**Go:**
- `os.ReadFile(...)`, `os.Open(...)`, `ioutil.ReadFile(...)`

**JavaScript:**
- `fs.readFileSync(...)`, `fs.createReadStream(...)`, `fs.readFile(...)`

### 2.2 File Write (file_write)

Same APIs but for write/append modes:
- `Files.write(...)`, `FileOutputStream(...)`, `Files.copy(...)` (if dest is writable)
- `open(file_path, 'w')`, `shutil.copy(...)`, `os.rename(...)` (can overwrite)
- `fs.writeFileSync(...)`, `fs.createWriteStream(...)`

**File delete:**
- `Files.delete(...)`, `File.delete()`, `os.remove(...)`, `fs.unlinkSync(...)`

---

## 3. Remote Calls

### 3.1 HTTP/REST Clients (rpc_call)

**Java:**
- `RestTemplate.getForObject(...)`, `RestTemplate.postForObject(...)`, `RestTemplate.exchange(...)`
- `WebClient.get().uri(...).retrieve()`, `WebClient.post().uri(...)`
- `HttpClient.send(...)`, `HttpURLConnection`, `OkHttpClient.newCall(...)`
- `@FeignClient` annotated interfaces (Spring Cloud)
- `Retrofit` service calls

**Python:**
- `requests.get(...)`, `requests.post(...)`, `httpx.get(...)`
- `urllib.request.urlopen(...)`

**Go:**
- `http.Get(...)`, `http.Post(...)`, `http.Client.Do(...)`

**JavaScript:**
- `fetch(...)`, `axios.get(...)`, `axios.post(...)`, `got(...)`, `node-fetch`

### 3.2 RPC Frameworks (rpc_call)

- Dubbo: `@DubboReference` calls
- gRPC: stub method calls (`stub.getOrder(...)`)
- Thrift: client calls
- SOAP: WebService calls
- Message queue send: `kafkaTemplate.send(...)`, `rabbitTemplate.convertAndSend(...)`

---

## 4. Network / SSRF (network_connect)

- `URL.openConnection()`, `URL.openStream()`, `URLConnection.connect()`
- `Socket(...)`, `new Socket(host, port)`
- `InetSocketAddress(...)` + `Socket.connect(...)`
- `requests.get(userProvidedUrl)` — SSRF if URL is user-controlled
- `curl_exec($ch)` (PHP) — SSRF if URL is user-controlled

---

## 5. Approval / Workflow Operations (approval_flow)

These are authorization-relevant because they implement DELAYED AUTHORIZATION:

- Workflow initiation: `workflowService.start(...)`, `approvalService.create(...)`, `ticketRepo.save(...)`
- Workflow state transition: `workflowService.approve(...)`, `approvalService.complete(...)`
- Workflow query: `approvalService.hasPendingApproval(...)`, `workflowService.getStatus(...)`

**Key insight:** If an operation goes through an approval flow, it IS authorized — the approval flow IS the auth mechanism. Do NOT flag operations behind an approval flow as BOLA just because there's no direct anchor binding. The approval flow provides delayed authorization.

---

## 6. Mass Assignment Operations (mass_assignment)

**Java:**
- `BeanUtils.copyProperties(source, target)`
- `BeanUtils.copyProperties(source, target, ignoreProperties)`
- `ConvertUtils.convert(...)` for bulk mapping
- `ModelMapper.map(source, Target.class)`
- `MapStruct` generated mapper methods (usually safe if mapper is hand-written)
- ORM `save()` with entity built from request body without field filtering

**Python:**
- `setattr(obj, key, value)` in a loop over request data
- Django: `Model.objects.create(**request.POST)` or `Model.objects.update(**request.data)`
- Flask: `model.populate_from_request(request.json)`

**Go:**
- `structs.Map(src).To(dst)` — struct mapping libraries
- `json.Unmarshal(requestBody, &entity)` followed by `db.Save(&entity)`

**JavaScript:**
- `Object.assign(target, req.body)`
- `{ ...entity, ...req.body }` spread operator in writes
- Sequelize: `Model.update(req.body, ...)`

---

## 7. Authorization Decision Operations (auth_decision)

These operations MAKE authorization decisions — their logic determines whether access is granted:

- `hasPermission(userId, resourceId, action)`
- `canAccess(userId, resourceId)`
- `isOwner(userId, resourceId)`
- `checkRole(userId, role)`
- Any method containing `@PreAuthorize`, `@PostAuthorize` — these are DECLARATIVE auth decisions
- Framework-specific: `request.isUserInRole(role)`, `@RolesAllowed`

**Important:** When analyzing an `auth_decision` sink, trace WHERE its inputs come from and whether the decision logic has a complete threat model. An auth_decision that only checks SOME conditions may create a false sense of security.

---

## Sink Priority for Analysis

When multiple datasinks exist, prioritize:
1. Direct data access sinks (db_read, db_write, file_read, file_write) — these are the PRIMARY concern
2. Remote access sinks (rpc_call, network_connect) — can amplify impact
3. Mass assignment sinks — can bypass intended access controls
4. Approval sinks — must verify completeness
5. Auth decision sinks — must verify correctness

## Cross-Language Pattern Summary

Regardless of language, look for the SEMANTIC meaning, not the syntax:
- Any code that fetches data from persistence → db_read
- Any code that modifies persistence → db_write
- Any code that reads from filesystem → file_read
- Any code that writes to filesystem → file_write
- Any code that makes outbound network requests → rpc_call or network_connect
- Any code that bulk-binds user input to an object → mass_assignment
- Any code that checks permissions → auth_decision
