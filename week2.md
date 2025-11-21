```rust
use std::{fmt::Display};

struct Calculator<X, Y> {x: X, y: Y}

trait AdditiveOperations {
    type Output;

    fn add(&self) -> Self::Output;

    fn sub(&self) -> Self::Output;
}

impl<X, Y, O> AdditiveOperations for Calculator<X, Y>
where
    Y: Copy,
    X: Copy + std::ops::Add<Y, Output = O> + std::ops::Sub<Y, Output = O> {
    type Output = O;

    fn add(&self) -> Self::Output {
        self.x + self.y
    }

    fn sub(&self) -> Self::Output {
        self.x - self.y
    }
}

trait MultiplicativeOperations {
    type Output;

    fn mul(&self) -> Self::Output;

    fn div(&self) -> Self::Output;
}

impl<X, Y, O> MultiplicativeOperations for Calculator<X, Y>
where
    Y: Copy,
    X: Copy + std::ops::Mul<Y, Output = O> + std::ops::Div<Y, Output = O> {
    type Output = O;
    
    fn mul(&self) -> Self::Output {
        self.x * self.y
    }
    
    fn div(&self) -> Self::Output {
        self.x / self.y
    }
}

impl<X, Y> std::fmt::Display for Calculator<X, Y>
where
    X: std::fmt::Display,
    Y: std::fmt::Display,
    Calculator<X, Y>: AdditiveOperations<Output: Display>,
    Calculator<X, Y>: MultiplicativeOperations<Output: Display> {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "Calculator({}, {})\n", self.x, self.y)?;
        write!(f, "Addition: {}\n", self.add())?;
        write!(f, "Subtraction: {}\n", self.sub())?;
        write!(f, "Multiplication: {}\n", self.mul())?;
        write!(f, "Division: {}\n", self.div())
    }
}

fn main() {
    let calculator = Calculator{
        x: 1,
        y: 2
    };
    println!("calculator: {}", calculator); 
}
```