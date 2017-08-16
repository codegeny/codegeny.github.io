---
layout: post
title: The only JUnit @Rule you will ever need
author: xavier
tag:
  - java
  - junit
---
Aren't you tired of those JUnit failing tests?
Don't you want to go home instead of fixing JUnit tests failing because of the last big re-factoring you have been working on the whole week?
Fear not, there is a solution: the ultimate JUnit rules that will make your tests pass even if they fail!

```java
import org.junit.rules.TestRule;
import org.junit.runner.Description;
import org.junit.runners.model.Statement;

public class SuccessRule implements TestRule {
	
	private static class SuccessStatement extends Statement {
		
		private final Statement base;
		
		public SuccessStatement(Statement base) {
			this.base = base;
		}

		@Override
		public void evaluate() throws Throwable {
			try {
				base.evaluate();
			} catch (Throwable thowable) {
				// Don't mention it!
			}
		}
	}

	@Override
	public Statement apply(Statement base, Description description) {
		return new SuccessStatement(base);
	}
}
```

```java
import org.junit.Assert;
import org.junit.Rule;
import org.junit.Test;

public class SuccessTest {
	
	@Rule
	public final SuccessRule successRule = new SuccessRule();
	
	@Test
	public void failingTestButNotReally() {
		Assert.fail("I am a failure");
	}
}
```

![JUnit success rule]({{ site.url }}/assets/junit-success-rule.png)

Don't thank me!