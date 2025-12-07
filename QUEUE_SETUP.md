# BullMQ Queue System Setup

✅ Successfully installed and configured BullMQ with Redis!

## What Was Installed

1. **Dependencies:**
   - `@nestjs/bullmq` - NestJS integration for BullMQ
   - `bullmq` - Modern Redis-based queue library
   - `ioredis` - Redis client for Node.js
   - `@bull-board/nestjs`, `@bull-board/api`, `@bull-board/express` - Queue monitoring dashboard

2. **Redis Configuration:**
   - Added `REDIS_HOST` and `REDIS_PORT` to [.env](backend/.env)
   - Default: `localhost:6379`

3. **Queue Module:**
   - Created [queues module](backend/src/queues/queues.module.ts) with notifications queue
   - [Notifications processor](backend/src/queues/notifications.processor.ts) handles background jobs
   - [Notifications service](backend/src/queues/notifications.service.ts) for adding jobs to queue
   - [Queue controller](backend/src/queues/queues.controller.ts) with example endpoints

## Before Starting the Server

**Install and start Redis:**

```bash
# macOS
brew install redis
brew services start redis

# Or using Docker
docker run -d -p 6379:6379 redis:alpine
```

## Usage

### 1. Start the Server

```bash
cd backend
npm run start:dev
```

### 2. Access Bull Board Dashboard

Open your browser:
```
http://localhost:3000/queues
```

This dashboard shows:
- Active jobs
- Waiting jobs
- Completed jobs
- Failed jobs
- Job details and logs

### 3. Test the Queue API

**Add a notification job:**
```bash
curl -X POST http://localhost:3000/queues/notifications \
  -H "Content-Type: application/json" \
  -d '{
    "userId": "123",
    "type": "email",
    "message": "Test notification",
    "metadata": {"subject": "Hello"}
  }'
```

**Schedule a delayed notification:**
```bash
curl -X POST http://localhost:3000/queues/notifications/scheduled \
  -H "Content-Type: application/json" \
  -d '{
    "data": {
      "userId": "123",
      "type": "sms",
      "message": "Delayed notification"
    },
    "delayInMs": 10000
  }'
```

**Get queue metrics:**
```bash
curl http://localhost:3000/queues/metrics
```

## Using in Your Code

Inject the `NotificationsService` anywhere in your app:

```typescript
import { NotificationsService } from './queues/notifications.service';

@Injectable()
export class ProductsService {
  constructor(private notificationsService: NotificationsService) {}

  async createProduct(product: Product) {
    // Create product...

    // Send async notification
    await this.notificationsService.sendNotification({
      userId: product.createdBy,
      type: 'email',
      message: 'Product created successfully!',
      metadata: { productId: product.id }
    });
  }
}
```

## Queue Features

✅ **Automatic retries** - Failed jobs retry 3 times with exponential backoff
✅ **Job scheduling** - Delay jobs to run in the future
✅ **Job priorities** - Higher priority jobs run first
✅ **Persistence** - Jobs survive server restarts
✅ **Monitoring** - Visual dashboard with Bull Board
✅ **Metrics** - Track queue performance
✅ **Rate limiting** - Control job processing rate
✅ **Concurrency** - Process multiple jobs in parallel

## Creating New Queues

See [Queue Documentation](backend/src/queues/README.md) for detailed instructions.

## Production Considerations

1. **Redis in Production:**
   - Use Redis Cluster or managed service (AWS ElastiCache, Redis Cloud, etc.)
   - Update `REDIS_HOST` and `REDIS_PORT` in production `.env`
   - Consider adding `REDIS_PASSWORD` for security

2. **Queue Security:**
   - Protect Bull Board dashboard with authentication
   - Use environment-based access control

3. **Monitoring:**
   - Set up alerts for failed jobs
   - Monitor queue size and processing time
   - Use Bull Board or integrate with monitoring tools (Datadog, New Relic, etc.)

4. **Scaling:**
   - Add more worker processes for high load
   - Use job concurrency settings
   - Implement rate limiting to protect external services
