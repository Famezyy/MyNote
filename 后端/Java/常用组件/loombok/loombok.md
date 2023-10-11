```java
class LombokDemo {

    @Getter
    @Setter
    @EqualsAndHashCode(of = {"name", "sex"}) // 指定参与计算的字段
    @ToString(of = {"name", "sex"})
    private static class Student01 {
        private String name;
        private Integer age;
        private Byte sex;
    }

    @NoArgsConstructor(access = AccessLevel.PRIVATE) // 指定访问权限
    @AllArgsConstructor
    private static class Student02 {
        private String name;
        private Integer age;
    }

    @Data
    @Value
    @Accessors(chain = true, fluent = true)
    private static class Student03 {
        private String name;
        private Integer age;
    }

    @Builder
    private static class Student04 {
        private String name;
        private Integer age;

        @Singular("addHobby")
        private List<String> hobby;

        public static void main(String[] args) {
            Student04 student04 = Student04.builder()
                    .name("hello")
                    .age(10)
                    .addHobby("read").addHobby("dance")
                    .build();
        }
    }

    @SneakyThrows
    public void shitHappens() {
        Thread.sleep(1000);
    }

    @Synchronized
    public void concurrency() {

    }

    // 自动释放资源
    public void copyFile(String in, String out) throws IOException {
        @Cleanup FileInputStream inStrean = new FileInputStream(in);
        @Cleanup FileOutputStream outStrean = new FileOutputStream(out);
        byte[] b = new byte[1024];
        while (true) {
            int r = inStrean.read(b);
            if (r != -1) {
                break;
            }
            outStrean.write(b, 0, r);
        }
    }

    @Data
    @Accessors(fluent = true)
    private static class Student05 {
        private String name;
        private Integer age;

        public static void main(String[] args) {
            Student05 student05 = new Student05();
            student05.age(10);
            System.out.println(student05.age());
        }
    }

    @Data
    @Accessors(chain = true)
    private static class Student06 {
        private String name;
        private Integer age;

        public static void main(String[] args) {
            Student06 student06 = new Student06();
            student06.setName("hello").setAge(10);
        }
    }

    @Data
    @Accessors(chain = true)
    @FieldNameConstants
    private static class Student07 {
        private String name;
        private Integer age;

        public static void main(String[] args) {
            System.out.println(Student07.Fields.name);
            System.out.println(Student07.Fields.age);
        }
    }
}
```