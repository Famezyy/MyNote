# Either

```java
public class Either<L, R> {

    /**
     * exception case
     */
    private L left;

    /**
     * normal case
     */
    private R right;

    public L getLeft() {
        return left;
    }

    public void setLeft(L left) {
        this.left = left;
    }

    public R getRight() {
        return right;
    }

    public void setRight(R right) {
        this.right = right;
    }

    public boolean isLeft() {
        return left != null;
    }

    public boolean isRight() {
        return right != null;
    }

    public static <L, R> Either<L, R> left(L exception) {
        Either<L, R> e = new Either<>();
        e.left = exception;
        return e;
    }

    public static <L, R> Either<L, R> right(R value) {
        Either<L, R> e = new Either<>();
        e.right = value;
        return e;
    }

    public <T> Either<L, T> mapRight(Function<R, T> function) {
        if (isLeft()) {
            return left(left);
        } else {
            return right(function.apply(right));
        }
    }

    /**
     * 将 EitherList 封装为一个 Either 对象
     * @param eitherList：either 的 list
     * @param accumulator：用于将两个异常值进行累加
     * @param fastFails：是否遇到第一个异常即停止并抛出该异常
     * @param <L>
     * @param <R>
     * @return：封装后的 Either 对象
     * @throws Exception
     */
    public static <L, R> Either<L, List<R>> sequence(List<Either<L, R>> eitherList, BinaryOperator<L> accumulator, boolean fastFails) throws Exception {
        if (eitherList.stream().allMatch(Either::isRight)) {
            return right(eitherList.stream().map(Either::getRight).collect(Collectors.toList()));
        }
        if (fastFails) {
            return left(eitherList.stream().filter(Either::isLeft).findFirst().orElseThrow(Exception::new).getLeft());
        }
        return left(eitherList.stream().filter(Either::isLeft).map(Either::getLeft).reduce(accumulator).orElseThrow(Exception::new));
    }
}
```

测试

```java
public class EitherTest {
    public static void main(String[] args) throws Exception {
        /*
        普通做法
        try {
            List<Member> collect = Stream.iterate(1, i -> i + 1)
                    .limit(100)
                    .map(i1 -> {
                        try {
                            return readLine(i1);
                        } catch (Exception e) {
                            throw new RuntimeException(e);
                        }
                    }).collect(Collectors.toList());
            for (Member member : collect) {
                System.out.println(member);
            }
        } catch (Exception e) {
            System.out.println(e.getMessage());
        }*/

        // 使用 Either 类
        List<Either<Exception, Member>> members = Stream.iterate(1, i -> i + 1)
            .limit(100)
            .map(EitherTest::readLineInEither).collect(Collectors.toList());
        Either<Exception, List<Member>> either = Either.sequence(members, (s1, s2) -> new Exception(s1.getMessage() + "\n" + s2.getMessage()), false);
        if (either.isLeft()) {
            System.out.println(either.getLeft().toString());
        } else {
            for (Member member : either.getRight()) {
                System.out.println(member);
            }
        }
    }

    static Member readLine(int i) throws Exception {
        if (new Random().nextInt(100) <= 50) {
            return new Member("张三", 20, 1);
        } else {
            throw new Exception("第" + i + "行出现错误");
        }
    }

    static Either<Exception, Member> readLineInEither(int i) {
        if (new Random().nextInt(100) <= 50) {
            return Either.right(new Member("张三", 20, 1));
        } else {
            return Either.left(new Exception("第" + i + "行出现错误"));
        }
    }
}
```

