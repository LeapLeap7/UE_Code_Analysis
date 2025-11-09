
GAS核心组件
> A component to easily interface with the 3 aspects of the AbilitySystem:
> 
> GameplayAbilities:
> - Provides a way to give/assign abilities that can be used (by a player or AI for example)
> - Provides management of instanced abilities (something must hold onto them)
> - Provides replication functionality
> - Ability state must always be replicated on the UGameplayAbility itself, but UAbilitySystemComponent provides RPC 
  replication for the actual activation of abilities
>
> GameplayEffects:
> - Provides an FActiveGameplayEffectsContainer for holding active GameplayEffects
> - Provides methods for applying GameplayEffects to a target or to self
> - Provides wrappers for querying information in FActiveGameplayEffectsContainers (duration, magnitude, etc)
> - Provides methods for clearing/remove GameplayEffects
>
> GameplayAttributes
> - Provides methods for allocating and initializing attribute sets
> - Provides methods for getting AttributeSets

# Attributes 属性

## UAttributeSet

### 成员变量：SpawnedAttributes

AbilitySystemComponent中存储的属性集合。

为了同步用了TArray，不然TMap会好点。
```c++
UPROPERTY(Replicated, ReplicatedUsing = OnRep_SpawnedAttributes, Transient)
TArray<TObjectPtr<UAttributeSet>>	SpawnedAttributes;
```

## 获取属性数据

### 获取指定类型的AttributeSet

遍历找有点耗，但基于AttributeSet不应该过多，所以可以接受。

这里可以看出UAttributeSet类型不应该重复。
```c++
const UAttributeSet* UAbilitySystemComponent::GetAttributeSubobject(const TSubclassOf<UAttributeSet> AttributeClass) const
{
	for (const UAttributeSet* Set : GetSpawnedAttributes())
	{
		if (Set && Set->IsA(AttributeClass))
		{
			return Set;
		}
	}
	return nullptr;
}
```

### 获取指定Attribute

无法获取Client端的预测数据。

考虑增加获取预测数据的能力：
- 给FGameplayAttribute增加PredictedValue，PredictionSourceData
- 给函数加个bPredicted的入参。

TODO: 单开个md描述预测设计

```c++
float UAbilitySystemComponent::GetGameplayAttributeValue(FGameplayAttribute Attribute, OUT bool& bFound) const
{
	// validate the attribute
	if (Attribute.IsValid())
	{
		// get the associated AttributeSet
		const UAttributeSet* InternalAttributeSet = GetAttributeSubobject(Attribute.GetAttributeSetClass());

		if (InternalAttributeSet)
		{
			// NOTE: this is currently not taking predicted gameplay effect modifiers into consideration, so the value may not be accurate on client
			bFound = true;
			return Attribute.GetNumericValue(InternalAttributeSet);
		}
	}

	// the attribute was not found
	bFound = false;
	return 0.0f;
}
```

## 修改属性数据

### 初始化

### 初始化AttributeSet

可以直接在Component里配：
```c++
UPROPERTY(EditAnywhere, Category="AttributeTest")
TArray<FAttributeDefaults>	DefaultStartingData;
```

配在DataTable里：
```c++
/** Used to initialize default values for attributes */
USTRUCT()
struct FAttributeDefaults
{
GENERATED_USTRUCT_BODY()

	FAttributeDefaults()
		: DefaultStartingTable(nullptr)
	{ }

	UPROPERTY(EditAnywhere, Category = "AttributeTest")
	TSubclassOf<UAttributeSet> Attributes;

	UPROPERTY(EditAnywhere, Category = "AttributeTest")
	TObjectPtr<UDataTable> DefaultStartingTable;
};
```

RowName对应AttributeSetName.PropertyName。

属性可以是纯数字，也可以自己定义一个FGameplayAttributeData结构，做特殊的计算功能。

UE官方建议用FGameplayAttributeData，不要纯数字。这样后续确实更好扩展。纯数字后面要加功能很麻烦。

```c++
void UAttributeSet::InitFromMetaDataTable(const UDataTable* DataTable)
{
    static const FString Context = FString(TEXT("UAttribute::BindToMetaDataTable"));

	for( TFieldIterator<FProperty> It(GetClass(), EFieldIteratorFlags::IncludeSuper) ; It ; ++It )
	{
		FProperty* Property = *It;
		FNumericProperty *NumericProperty = CastField<FNumericProperty>(Property);
		if (NumericProperty)
		{
			FString RowNameStr = FString::Printf(TEXT("%s.%s"), *Property->GetOwnerVariant().GetName(), *Property->GetName());
		
			FAttributeMetaData * MetaData = DataTable->FindRow<FAttributeMetaData>(FName(*RowNameStr), Context, false);
			if (MetaData)
			{
				void *Data = NumericProperty->ContainerPtrToValuePtr<void>(this);
				NumericProperty->SetFloatingPointPropertyValue(Data, MetaData->BaseValue);
			}
		}
		else if (FGameplayAttribute::IsGameplayAttributeDataProperty(Property))
		{
			FString RowNameStr = FString::Printf(TEXT("%s.%s"), *Property->GetOwnerVariant().GetName(), *Property->GetName());

			FAttributeMetaData * MetaData = DataTable->FindRow<FAttributeMetaData>(FName(*RowNameStr), Context, false);
			if (MetaData)
			{
				FStructProperty* StructProperty = CastField<FStructProperty>(Property);
				check(StructProperty);
				FGameplayAttributeData* DataPtr = StructProperty->ContainerPtrToValuePtr<FGameplayAttributeData>(this);
				check(DataPtr);
				DataPtr->SetBaseValue(MetaData->BaseValue);
				DataPtr->SetCurrentValue(MetaData->BaseValue);
			}
		}
	}

	PrintDebug();
}
```

### 添加AttributeSet

AttributeSet会跟随Component作为SubObject被同步。添加时标脏。
```c++
void UAbilitySystemComponent::AddSpawnedAttribute(UAttributeSet* Attribute)
{
    if (!IsValid(Attribute))
    {
        return;
    }

	if (SpawnedAttributes.Find(Attribute) == INDEX_NONE)
	{
		if (IsUsingRegisteredSubObjectList() && IsReadyForReplication())
		{
			AddReplicatedSubObject(Attribute);
		}

		SpawnedAttributes.Add(Attribute);
		SetSpawnedAttributesListDirty();
	}
}
```


