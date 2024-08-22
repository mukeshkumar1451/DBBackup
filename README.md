   public static String getISTFormatDate(Date date) {
//    Date utcDateString = new Date();
        DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm");
        Instant utcInstant = date.toInstant();
        ZonedDateTime zonedDateTime = utcInstant.atZone(ZoneId.of("Asia/Kolkata"));
        String formattedDate = formatter.format(zonedDateTime);
        return formattedDate;
    }
