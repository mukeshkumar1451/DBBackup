public static void main(String[] args) {

    String inputDate = "2024-08-22 23:55";
    DateTimeFormatter inputFormatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm");
    LocalDateTime localDateTime = LocalDateTime.parse(inputDate, inputFormatter);
    ZonedDateTime outputDateTime = ZonedDateTime.of(localDateTime, ZoneId.of("UTC"));
    DateTimeFormatter outputFormatter = DateTimeFormatter.ofPattern("yyyy-MM-dd'T'HH:mm:ss.SSSSSS'Z'").withZone(ZoneId.of("UTC"));
    String outputDate = outputFormatter.format(outputDateTime);
    System.out.println(outputDate);

}
