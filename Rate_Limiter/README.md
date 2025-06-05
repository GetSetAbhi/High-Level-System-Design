# Rate Limiter Algorithms

---

```java
interface Ratelimiter {
	boolean check();
}

class TokenBucketRatelimiter implements Ratelimiter {
	
	private int maxTokens;
	private long refreshRate;
	private int currentToken;
	private long lastTokenUseTime;

	public TokenBucketRatelimiter(int maxTokens, int refreshRate) {
		this.maxTokens = maxTokens;
		this.refreshRate = refreshRate / 1000;
		this.currentToken = this.maxTokens;
		this.lastTokenUseTime = System.currentTimeMillis();
	}
	
	private void refresh() {
		long currentTime = System.currentTimeMillis();
		long diff = currentTime - this.lastTokenUseTime;
		int newTokens = (int) (diff * this.refreshRate);
		this.currentToken = Math.min(maxTokens, this.currentToken + newTokens);
		this.lastTokenUseTime = currentTime;
	}

	@Override
	public boolean check() {
		this.refresh();
		if (currentToken > 0 && this.currentToken <= this.maxTokens ) {
			this.currentToken -= 1;
			return true;
		}
		return false;
	}
	
}

class SlidingWindowRatelimiter implements Ratelimiter {

	private Queue<Long> queue;
	private int maxTokens;
	private long window;
	
	public SlidingWindowRatelimiter(int maxTokens, long window) {
		queue = new LinkedList<>();
		this.maxTokens = maxTokens;
		this.window = window;
	}

	@Override
	public boolean check() {
		long currentTime = System.currentTimeMillis();
		
		while (queue.size() > 0 && currentTime - queue.peek() > this.window) {
			queue.poll();
		}
		
		if (this.queue.size() < maxTokens) {
			this.queue.add(currentTime);
			return true;
		}
		return false;
	}
}
```