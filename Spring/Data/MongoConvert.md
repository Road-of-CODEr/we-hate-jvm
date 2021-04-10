# Mongo 특정 field Convert 삽질
MongoDB에 어떤 객체를 넣어야 되는데, 이때 그 객체의 특정 field만 암복호화 처리를 해야된다.
그리고 원래 format 그대로 저장 및 조회가 가능해야한다.

## 삽질을 하게된 계기
JPA는 AttributeConverter라는 녀석이 있어서 매우 간단하고 변환이 가능하다! 하지만 Mongo는 JPA를 사용하지 않기때문에 이걸 사용할 수 없음
* 아래와 같이하면 자동으로 됨
```kotlin
import javax.persistence.AttributeConverter
import javax.persistence.Converter

@Converter(autoApply = true)
class EmailAddressConverter : AttributeConverter<EmailAddress, String> {
    override fun convertToDatabaseColumn(attribute: EmailAddress?): String? {
        return CryptoUtils.encrypt(attribute?.value)
    }

    override fun convertToEntityAttribute(dbData: String?): EmailAddress? {
        return dbData?.let { EmailAddress(CryptoUtils.decrypt(it)!!) }
    }
}

class EmailAddress(val value: String)
```

오! Mongo에도 CustomConversion이라는게 있네? 자 그럼 원래 format 그대로 어떻게 하지 

## 삽질
아 일단 Document Layer까지 내려가서 걍 다 만들자<br>
아니 근데 이러면 새로운거 생길 때마다 다 만들어줘야되잖아 극혐이네
```kotlin
@Document(collection = "person")
data class Person(
    @Id
    val id: String = "",
    val personId: Long,
    var 이름: String,
    var 주민등록번호: String,
)
```
```kotlin
import org.bson.Document
import org.springframework.core.convert.converter.Converter
import org.springframework.data.convert.ReadingConverter
import org.springframework.data.convert.WritingConverter
import org.springframework.stereotype.Component

@Component
@ReadingConverter
class 주민등록번호ReadConverter : Converter<Document, Person> {
    override fun convert(source: Document): Person? {
        var personId = source["personId"] as Long
        var 이름 = source["이름"] as String
        var 주민등록번호 = source["주민등록번호"] as String
        return Person("", personId, 이름, CryptUtils.decrypt(주민등록번호))
    }
}

@Component
@WritingConverter
class 주민등록번호WriteConverter : Converter<Person, Document> {
    override fun convert(source: Person): Document? {
        val document = Document()
        document["personId"] = source.personId
        document["이름"] = source.이름
        document["주민등록번호"] = CryptUtils.encrypt(source.주민등록번호)
        return document
    }
}
```

## 결론
Serializer, Deserializer, Converter를 쓰면 가능하다...
```kotlin
@Document(collection = "person")
data class Person(
    @Id
    val id: String = "",
    val personId: Long,
    var 이름: String,
    var 주민등록번호: EncMongoString,
)
```
```kotlin
@Configuration
class JacksonConfiguration {
    @Primary
    @Bean
    fun objectMapper(): ObjectMapper {
        return jacksonObjectMapper()
            .registerModule(JavaTimeModule())
            .registerModule(
                SimpleModule()
                    .addSerializer(EncMongoString::class.java, encMongoStringSerializer())
                    .addDeserializer(EncMongoString::class.java, encMongoStringDeserializer())
            )
    }
}

class EncMongoString(val value: String)

fun encMongoStringSerializer() = object : JsonSerializer<EncMongoString>() {
    override fun serialize(value: EncMongoString, gen: JsonGenerator, serializers: SerializerProvider) {
        gen.writeString(value.value)
    }
}

fun encMongoStringDeserializer() = object : JsonDeserializer<EncMongoString>() {
    override fun deserialize(p: JsonParser, ctxt: DeserializationContext?): EncMongoString {
        return EncMongoString(p.text)
    }
}
```
```kotlin
@Configuration
class MongoConfiguration {

    @Bean
    fun customConversions(): MongoCustomConversions {
        val converters = listOf(
            EncMongoStringToStringConverter(),
            StringToEncMongoStringConverter()
        )
        return MongoCustomConversions(converters)
    }

    internal class EncMongoStringToStringConverter : Converter<EncMongoString, String> {
        override fun convert(source: EncMongoString): String? {
            return CryptUtils.encrypt(source.value)
        }
    }

    internal class StringToEncMongoStringConverter : Converter<String, EncMongoString> {
        override fun convert(source: String): EncMongoString {
            return EncMongoString(CryptUtils.decrypt(source))
        }
    }

}
```
* 저장할 때는 Deserialize 후, Convert로 암호화 되어 저장!
* 조회할 때는 Convert 후, Serialize!

이러면 원래 format을 바꾸지 않고 특정 field를 암복호화 할 수 있다.

## 끝

[Mongo-ex](https://github.com/Spring-Data-Things/mongodb-ex)
