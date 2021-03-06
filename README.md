# FSM
## Finite State Machine


![moro logo](moro_logo.jpg)
### Об одном способе реализации конечного автомата.
   Идея состоит в том, что реализация каждого режима работы устройства должна производиться независимо от иных режимов, но использовать некий общий инструментарий, обеспечивающий безаварийный доступ к ресурсам прибора. Как говорил великий Дейкстра: "Программирование это поиск глубинной простоты в запутанном клубке сущностей." - не дословно, но близко к источнику. В свое время мне понадобилось не более двух месяцев, чтобы подавить в себе естественное отторжение и привести проект к виду, с которым комфортно и, главное безопасно работать. 

### Конечный автомат, он же Finite State Mashine (FSM).
Любой или почти любой процесс можно представить в виде последовательности шагов от одного состояния к другому. Разработчик мало чего стоит, если не представляет себе все возможные состояния, число которых конечно - именно поэтому автомат и называют "конечным". Вдохновением можно запастись [здесь](https://chipenable.ru/index.php/programming-avr/item/90-realizatsiya-konechnogo-avtomata-state-machine.html) . Способ несомненно хорош. Но оцените предложенный ниже.

Как же будет выглядеть реализация конкретного режима, например простого заряда аккумуляторной батареи? Графически режим такого заряда может быть представлен так: идите от жирной точки вверху, в цветных бланках короткие примечания к состояниям.
![Заряд CC/CV](CC_CV_fsm.jpg)
Вырисовывается не так уж и много состояний - чуть более десятка. Слева - состояния ввода параметров с возможностью отказа от дальнейшего ввода и перехода на исполнение или выход. Справа - последовательность шагов заряда - подъем тока, удержание тока и удержание напряжения. В центре - состояние отложенного старта.
Оформим каждое состояние соответствующим объявлением класса в общем поле имен CcCvFsm. Здесь же можно определить необходимые константы, свойственные для этого режима, чтобы всегда были "под рукой" в структурах.

##### файл cccvfsm.h
```c++
#ifndef _CCCVFSM_H_
#define _CCCVFSM_H_

#include "mstate.h"

namespace CcCvFsm    // Поле имен для режима простого заряда
{
  struct MChConsts
  {
      // Пределы регулирования
      static constexpr float i_l =  0.2f;
      static constexpr float i_h = 12.2f;
      static constexpr float v_l = 10.0f;
      static constexpr float v_h = 16.0f;
  };

  struct MPidConstants
  {
      // Параметры регулирования
      static constexpr float outputMin            = 0.0f;
      static constexpr float outputMaxFactor      = 1.05f;     // factor for current limit
  };

  class MStart : public MState
  {      
    public:
      MStart(MTools * Tools);
      MState * fsm() override;
  };

  class MSetCurrentMax : public MState
  {
    public:  
      MSetCurrentMax(MTools * Tools);
      MState * fsm() override;
  };
 
  class MSetVoltageMax : public MState
  {
    public:  
      MSetVoltageMax(MTools * Tools);
      MState * fsm() override;
  };

  class MSetCurrentMin : public MState
  {
    public:    
      MSetCurrentMin(MTools * Tools);
      MState * fsm() override;
  };

  class MSetVoltageMin : public MState
  {
    public:    
      MSetVoltageMin(MTools * Tools);
      MState * fsm() override;
  };

  class MPostpone : public MState
  {
    public:  
      MPostpone(MTools * Tools);
      MState * fsm() override;
  };

  class MUpCurrent : public MState
  {
    public:  
      MUpCurrent(MTools * Tools);
      MState * fsm() override;
  };

  class MKeepVmax : public MState
  {
    public:
      MKeepVmax(MTools * Tools);
      MState * fsm() override;
  };

  class MKeepVmin : public MState
  {
    public:  
      MKeepVmin(MTools * Tools);
      MState * fsm() override;
  };

  class MStop : public MState
  {
    public: 
      MStop(MTools * Tools);
      MState * fsm() override;
  };

  class MExit : public MState
  {
    public:
      MExit(MTools * Tools);
      MState * fsm() override;
  };

};

#endif  // !_CCCVFSM_H_
```
Не трудно заметить, что каждое состояние представлено соответствующим классом, а исполняемые функции - виртуальной функцией fsm().
Классы состояний режима, как и структуры с константами имеют свое поле имен, так что в других режимах могут использоваться те же имена, что, согласитесь, удобно.  А что за классы MState и MTools? Дойдем и до них. А пока оценим самодокументированность кода.

Перейдем к реализации. Определения всех состояний не приводятся, они однотипны. Рассмотрим на примере двух-трёх. Пока обращайте внимание только на содержательную часть, принимая "оболочку" как данность. По возможности используются говорящие имена. О классах достаточно знать лишь основы. Доверьтесь профессионалу - это при мне прямо "из-под волос" выдал программист, до которого нам далеко.
Честно признаюсь - меня поначалу стошнило))
 Пока просто скользните взглядом, не вчитывайтесь, ощутите возможность окинуть одним взглядом режим заряда от начала и до конца.

##### файл cccvfsm.cpp
```c++
#include "cccvfsm.h"
#include "mtools.h"
#include "mboard.h"
#include "mdisplay.h"
namespace CcCvFsm
{
  // Состояние "Старт", инициализация выбранного режима работы (Заряд CCCV).
  MStart::MStart(MTools * Tools) : MState(Tools)
  {
      // Параметры заряда из энергонезависимой памяти, Занесенные в нее при предыдущих включениях, как и
      // выбранные ранее номинальные параметры батареи (напряжение, емкость).
      // Tools->setVoltageMax( ... ); Tools->setVoltageMin( ... ); и т.п.
      
      // Индикация подсказки по строкам дисплея
      Display->getTextMode( (char*) "   CC/CV SELECTED    " );
      Display->getTextHelp( (char*) "  P-DEFINE  C-START  " );
      Display->progessBarOff();
  }
  MState * MStart::fsm()
  {
    switch ( Keyboard->getKey() )
    {
      case MKeyboard::C_CLICK :                // Если короткое нажатие на "C"
        // Пересчет из параметров батареи в напряжения и токи
        Tools->setVoltageMax( MChConsts::voltageMaxFactor * Tools->getVoltageNom() );
        Tools->setVoltageMin( MChConsts::voltageMinFactor * Tools->getVoltageNom() );
        Tools->setCurrentMax( MChConsts::currentMaxFactor * Tools->getCapacity() );
        Tools->setCurrentMin( MChConsts::currentMinFactor * Tools->getCapacity() );
        return new MPostpone(Tools);           // Выбран переход в состояние отложенного старта
      case MKeyboard::P_CLICK :                // Если короткое нажатие на "P"
        return new MSetCurrentMax(Tools);      // Выбран переход в состояние уточнения настроек заряда.
      default:;
    }
    Display->voltage( Board->getRealVoltage(), 2 ); // Во второй строке дисплея показывать напряжение
    Display->current( Board->getRealCurrent(), 1 ); // В первой строке дисплея показывать ток
    return this;                               // Ничего не выбрано, оставаться в этом состоянии до выбора
  };


  // Состояние "Коррекция максимального тока заряда"."
  MSetCurrentMax::MSetCurrentMax(MTools * Tools) : MState(Tools)
  {
    // Индикация подсказки
    Display->getTextMode( (char*) "U/D-SET CURRENT MAX" );
    Display->getTextHelp( (char*) "  B-SAVE  C-START  " );
  }
  MState * MSetCurrentMax::fsm()
  {
    switch ( Keyboard->getKey() )             // Что нажато и как долго
    {
      case MKeyboard::C_LONG_CLICK :
        return new MStop(Tools);              // Переход в состояние стоп
      case MKeyboard::C_CLICK :            
        return new MPostpone(Tools);          // Отказ от дальнейшего ввода параметров - исполнение
      case MKeyboard::B_CLICK :            
        Tools->saveFloat( MNvs::nCcCv, MNvs::kCcCvImax, Tools->getCurrentMax() ); 
        return new MSetVoltageMax(Tools);     // Сохранить и перейти к следующему параметру
      case MKeyboard::UP_CLICK :
      case MKeyboard::UP_AUTO_CLICK :
        Tools->currentMax = Tools->upfVal( Tools->currentMax, MChConsts::i_l, MChConsts::i_h, 0.1f );
        break;             // Корректировать по короткому нажатию на +0.1, по удержанию - повторять +0.1
      case MKeyboard::DN_CLICK :
      case MKeyboard::DN_AUTO_CLICK :
        Tools->currentMax = Tools->dnfVal( Tools->currentMax, MChConsts::i_l, MChConsts::i_h, 0.1f );
        break;
      default:;
    }
    // Если не закончили ввод, то индикация введенного
    Display->voltage( Board->getRealVoltage(), 2 );
    Display->current( Tools->getCurrentMax(), 1 );
    return this;                               // и остаемся в том же состоянии
  };


  //... несколько состояний аналогичны и опущены


  // Состояние: "Подъем и удержание максимального тока"
  MUpCurrent::MUpCurrent(MTools * Tools) : MState(Tools)
  {   
    // Индикация подсказки
    Display->getTextMode( (char*) " UP CURRENT TO MAX " );
    Display->getTextHelp( (char*) "       C-STOP      " );
    // Обнуляются счетчики времени и отданного заряда
    Tools->clrTimeCounter();
    Tools->clrAhCharge();
    // Включение (показано схематично, команды драйверу)
    Tools->setComAmp( ток );
    Tools->setComVolt( напряжение );
    Tools->setComGo();
  }     
  MUpCurrent::MState * MUpCurrent::fsm()
  {
    Tools->chargeCalculations();              // Подсчет отданных ампер-часов.
    // После пуска короткое нажатие кнопки "C" производит отключение тока.
    if(Keyboard->getKey(MKeyboard::C_CLICK)) { return new MStop(Tools); }    
    // Проверка напряжения и переход на поддержание напряжения.
    if( Board->getRealVoltage() >= Tools->getVoltageMax() ) { return new MKeepVmax(Tools); }
      
    // Индикация фазы подъема тока не выше заданного
    Display->voltage( Board->getRealVoltage(), 2 );
    Display->current( Board->getRealCurrent(), 1 );
    Display->progessBarExe( MDisplay::GREEN );
    Display->duration( Tools->getChargeTimeCounter(), MDisplay::SEC );
    Display->amphours( Tools->getAhCharge() );
      
    return this;
  };

  // Третья фаза заряда - достигнуто снижение тока заряда ниже заданного предела.
  // Проверки различных причин завершения заряда.
  MKeepVmin::MKeepVmin(MTools * Tools) : MState(Tools)
  {
    // Индикация подсказки
    Display->getTextMode( (char*) " KEEP VOLTAGE MIN  " );
    Display->getTextHelp( (char*) "       C-STOP      " );
    // Порог регулирования по напряжению (схематично)
    Tools->setComVoltMin();         
  }     
  MState * MKeepVmin::fsm()
  {
    Tools->chargeCalculations();        // Подсчет отданных ампер-часов.
    // Окончание процесса оператором.
    if (Keyboard->getKey(MKeyboard::C_CLICK)) 
    { return new MStop(Tools); }       
    // Здесь возможны проверки других условий окончания заряда
    // if( ( ... >= ... ) && ( ... <= ... ) )  { return new MStop(Tools); }
    // Максимальное время заряда, задается в "Настройках"
    if( Tools->getChargeTimeCounter() >= ( Tools->charge_time_out_limit * 36000 ) ) 
    { return new MStop(Tools); }
    Tools->setComVolt( указываем напряжение в милливольтах );           // Регулировка по напряжению
    Display->progessBarExe( MDisplay::MAGENTA );
    Display->duration( Tools->getChargeTimeCounter(), MDisplay::SEC );
    Display->amphours( Tools->getAhCharge() );
    return this;
  };


 // Состояние: "Завершение заряда"
  MStop::MStop(MTools * Tools) : MState(Tools)
  {
    Tools->shutdownCharge();
    Display->getTextHelp( (char*) "              C-EXIT " );
    Display->getTextMode( (char*) "   CC/CV CHARGE OFF  " );
    Display->progessBarStop();
  }    
  MState * MStop::fsm()
  {
    switch ( Keyboard->getKey() )
    {
      case MKeyboard::C_CLICK :
        return new MExit(Tools);
      default:;
      
      //Display->progessBarOff();
    }
    return this;
  };

  // Состояние: "Индикация итогов и выход из режима заряда в меню диспетчера" 
  MExit::MExit(MTools * Tools) : MState(Tools)
  {
    Tools->shutdownCharge();
    Display->getTextHelp( (char*) "              C-EXIT " );
    Display->getTextMode( (char*) "   CC/CV CHARGE OFF  " );
    Display->progessBarOff();
  }    
  MState * MExit::fsm()
  {
    switch ( Keyboard->getKey() )
    {
      case MKeyboard::C_CLICK :
        Display->getTextMode( (char*) "    CC/CV CHARGE     " );
        Display->getTextHelp( (char*) " U/D-OTHER  B-SELECT " );
        return nullptr;                             // Возврат к выбору режима
      default:;
    }
    return this; // до нажатия кнопки "С" удерживается индикация о продолжительности и отданном заряде.
  };
};
// !Конечный автомат режима простого заряда (CCCV).
```
Вот она! глубинная простота. Про всякие там "поздние связывания" нам знать не обязательно. Но то, что определение класса состояния начинается с конструктора (помним, что конструктор не возвращает никаких значений) - это инициализация состояния,  а далее следует определение объявленной ранее виртуальной функции, то есть что исполняется на этом шаге при каждом вызове.  Короче. Любое отдельно взятое состояние будет выглядеть понятно. Получится думать не "галактикой", а одним и только одним состоянием в каждый момент.
Есть одно "но", связанное с тем, что конструктор нового состояние создается до выхода из предыдущего. Не вдаваясь в объяснения - здесь так делать можно.
```c++
// Состояние: "..."

  MName::MMName(MTools * Tools) : MState(Tools)
  {   
    // Это конструктор, здесь инициализация состояния
    // Используя методы из MTools выполните то, что требуется исполнить при переходе в это
    // состояние из какого-то иного состояния. Учтите, что конструктор ничего не возвращает.
  }
    
  MName::MName * MName::fsm()
  {
    // Это определение и вызов функции, здесь описывается методами MTools что надо выполнять
    // вслед за инициализацией и при каждом вызове этого состояния операционной системой.
    // Если при каждом вызове состояния что-то безусловно меняется, например индикация,
    // то укажите здесь.

    // В порядке приоритета при необходимости приведите проверки условий, при выполнении
    // которых выполняется запрос перехода в иное выбранное состояние, например

    if( условие = true ) { return new MName2(Tools); }

    // Если ни одно из условий не выполняется, то запрашивается возврат в это же состояние.
    // Однако если в этом случае опять-таки что-то меняется, задаем  здесь.

    return this;
    // return nullptr;   // Или для выхода из последнего состояния к выбору иного режима
  };
```
Как вы заметили, выделенные строки одинаковы во всех состояниях, радость-то какая))

Определения состояний расположены в произвольном порядке. И связаны только через return this, return name или return nullptr, которые выполняются при следующем обращении операционной системы к задаче. Какой? Не всё сразу.

#### Теперь базовый класс MState:

##### файл mstate.h
```c++
#ifndef _MSTATE_H_
#define _MSTATE_H_

class MTools;
class MBoard;
class MDisplay;
class MKeyboard;

class MState
{
  public:
    MState(MTools * Tools);
    virtual ~MState(){}
    virtual MState * fsm() = 0;

  protected:
    MTools    * Tools    = nullptr;
    MBoard    * Board    = nullptr;
    MDisplay  * Display  = nullptr;
    MKeyboard * Keyboard = nullptr;
};

#endif // !_MSTATE_H_
```
Функция fsm() объявлена виртуальной и в файлах реализации режимов наследуется как виртуальная, поэтому там "virtual" не обязателен, да и "override" добавлен исключительно для читабельности.

файл mstate.cpp
```c++
#include "mstate.h"
#include "mtools.h"

MState::MState(MTools * Tools) :
  Tools(Tools),
  Board(Tools->Board),
  Display(Tools->Display),
  Keyboard(Tools->Keyboard) {}
```

Вы не находите, что это шедевр? Не мой - профессионала в этом деле.   ***Я не бездействовал, я сразу на капу нажал.***  
Попросил сделать максимально удобно из двух возможных вариантов для непрограммиста, но с навыками программирования.
Кстати, если интересует почему все имена классов начинаются на одну и ту же букву - так это наше, фамильное)).

Далее по порядку:
MTools - класс, где состедоточено большая часть инструментария. Предполагается, что это пишет разработчик аппаратной части проекта, хорошо осведомленный в том, что можно и за какие рамки заходить нельзя.
В отдельные классы оформлены (исторически так получилось)  описания и управление некоторыми ресурсами аппаратной части проекта - MBoard, MDisplay, MKeyboard. К ним вернусь позже.

На очереди класс MDispatcher, который отвечает за выбор режима работы прибора посредством меню. Но сначала в конструкторе - инициализация всего-всего.  Будете искать в Setup() привычные init(); - не ищите.

##### файл dispatcher.h
```c++
#ifndef _DISPATCHER_H_
#define _DISPATCHER_H_

class MTools;
class MBoard;
class MDisplay;
class MState;

class MDispatcher
{
  public:
    enum MODES
    {
      OPTIONS = 0,    // режим ввода настроек (не отключаемый)
      TEMPLATE,       // шаблон режима
      DCSUPPLY,       // режим источника постоянного тока
      PULSEGEN,       // режим источника импульсного тока
      CCCVCHARGE,     // режим заряда "постоянный ток / постоянное напряжение"
      PULSECHARGE,    // режим импульсного заряда
      RECOVERY,       // режим восстановления
      STORAGE,        // режим хранения
      DEVICE,         // режим заводских регулировок
      SERVICE         // режим Сервис АКБ
    };

  public:
    MDispatcher(MTools * tools);

    void run();
    void delegateWork();
    void textMode(int mode);

  private:
    MTools    * Tools;
    MBoard    * Board;
    MDisplay  * Display;
    MState    * State = 0;

    bool latrus = false;
    int mode = CCCVCHARGE;
};

#endif //_DISPATCHER_H_
```
 В начале, как и ранее было показано, идет конструктор класса. В данном случае выполняет роль инициализации всего-всего. Константы ... кто видел где константы??? Да где же им быть - в объявлении класса. Тут есть как свои достоинства, так и недостатки.


##### файл dispatcher.cpp
```c++
#include "mdispatcher.h"
#include "nvs.h"
#include "mtools.h"
#include "mboard.h"
#include "mkeyboard.h"
#include "mdisplay.h"

#include "modes/templatefsm.h"
// ...
#include "modes/cccvfsm.h"
#include "modes/servicefsm.h"

#include <string>

MDispatcher::MDispatcher(MTools * tools): Tools(tools), Board(tools->Board), Display(tools->Display)
{
    char sLabel[ MDisplay::MaxString ] = { 0 };
    strcpy( sLabel, "  OLMORO ** ELTRANS  " );
    Display->getTextLabel( sLabel );

    latrus = Tools->readNvsBool( MNvs::nQulon, MNvs::kQulonLocal, true );
    mode   = Tools->readNvsInt ( MNvs::nQulon, MNvs::kQulonMode, 0 );   // Индекс массива

    textMode( mode );
    Tools->powInd = Tools->readNvsInt  ( MNvs::nQulon, MNvs::kQulonPowInd, 3);
    // Индекс массива с набором батарей 3
    Tools->akbInd = Tools->readNvsInt  ( MNvs::nQulon, MNvs::kQulonAkbInd, 3);
    Tools->setVoltageNom( Tools->readNvsFloat( MNvs::nQulon, MNvs::kQulonAkbU, Tools->akb[3][0]));
    Tools->setCapacity( Tools->readNvsFloat( MNvs::nQulon, MNvs::kQulonAkbAh, Tools->akb[3][1]) );

    Tools->postpone = Tools->readNvsInt( MNvs::nQulon, MNvs::kQulonPostpone,  3 );

    // Калибровки измерителей 
    Board->voltageMultiplier  = Tools->readNvsFloat( MNvs::nQulon, MNvs::kQulonVmult,   1.00f );
    Board->voltageOffset      = Tools->readNvsFloat( MNvs::nQulon, MNvs::kQulonVoffset, 0.00f );
    Board->currentMultiplier  = Tools->readNvsFloat( MNvs::nQulon, MNvs::kQulonImult,   1.40f );
    Board->currentOffset      = Tools->readNvsFloat( MNvs::nQulon, MNvs::kQulonIoffset, 0.00f );
}

void MDispatcher::run()
{
  // Индикация при инициализации процедуры выбора режима работы
  Display->voltage( Board->getRealVoltage(), 2 );
  Display->current( Board->getRealCurrent(), 1 );

  // Выдерживается период запуска для вычисления амперчасов
  if (State)
  {
    // rabotaem so state mashinoj
    MState * newState = State->fsm();     
    if (newState != State)                  //state changed!
    {
      delete State;
      State = newState;
    }
    //esli budet 0, na sledujushem cikle uvidim
  }
  else //state ne opredelen (0) - vybiraem ili pokazyvaem rezgim
  {
    if (Tools->Keyboard->getKey(MKeyboard::UP_CLICK))
    {
      if (mode == (int)SERVICE) mode = OPTIONS;
      else mode++;
      textMode( mode );
    }

    if (Tools->Keyboard->getKey(MKeyboard::DN_CLICK))
    {
      if (mode == (int)OPTIONS) mode = SERVICE;
      else mode--;
      textMode( mode );
    }

    if (Tools->Keyboard->getKey(MKeyboard::B_CLICK))
    {
      // Запомнить крайний выбор режима
      Tools->writeNvsInt( MNvs::nQulon, "mode", mode );

      switch (mode)
      {
        case OPTIONS:     State = new OptionFsm::MStart(Tools);     break;
          // несколько режимов опущены
        case CCCVCHARGE:  State = new CcCvFsm::MStart(Tools);       break;
        case SERVICE:     State = new ServiceFsm::MStart(Tools);    break;
        default:          break;
      }
    } // !B_CLICK
  }
}

void MDispatcher::textMode(int mode)
{
  char sMode[ MDisplay::MaxString ] = { 0 };
  char sHelp[ MDisplay::MaxString ] = { 0 };

  switch(mode)
  {
    case OPTIONS:
      sprintf( sMode, "OPTIONS: BATT.SELECT," );
      sprintf( sHelp, "CALIBRATION,TIMER ETC" );
    break;

    case CCCVCHARGE:
      sprintf( sMode, "    CC/CV CHARGE:    " );
      sprintf( sHelp, " U/D-OTHER  B-SELECT " );
    break;

      // ...
    case SERVICE:
      sprintf( sMode, "  BATTERY SERVICE:   " );
      sprintf( sHelp, " ADJUSTING THE DEVICE" );
    break;

    default:
      sprintf( sMode, "  ERROR:             ");
      sprintf( sHelp, "  UNIDENTIFIED MODE  " );
    break;
  }

  Display->getTextMode( sMode );
  Display->getTextHelp( sHelp );
}
```
Диспетчер выполнят две функции: первая - работа с меню до запуска выбранного режима, вторая - инициализация состояния и вызов виртуальной функции, определенной в классе этого состояния. Функция run() диспетчера проверяет номер состояния, который возвращается активным состоянием на ноль (nullptr) - это означает выход из режима в меню выбора. При ненулевом значении и отличным номером инициируется новое состояние, иначе будет исполнена та же функция, что и при предыдущем вызове естественно без инициализации. Всё тривиально просто. Но исполнителя требуется в первую очередь аккуратность.
Большую часть диспетчера занимает отправка на дисплей строк меню. Какие-то фоновые функции можно выполнить и здесь, в диспетчере, но лучше под них выделить отдельную задачу для RTOS и сбагрить её кому-то.


Задачи RTOS. Хотите вы или нет, но с операционной системой реального времени Free RTOS придется подружиться. Разработчики Expressif постарались максимально облегчить жизнь программиста. Учебники по Free RTOS - в помощь, но надо иметь ввиду, что учебники писались для одноядерных процессоров, а ESP32 имеет два ядра. И блокировать, например, оба не есть хорошо. Рекомендуется для задач обслуживания радиотехнического блока использовать одно ядро, а для целевой программы - другое. Оформить сказанное не просто, а очень просто. Многое в настройках RTOS уже сделано за нас, тем более, если используется SDK от Expressif да ещё и под Ардуиной.  Уверяю, что скоро вы забудете, что ваш код исполняется под ОС. А вот без каких данных не обойтись - так это временные параметры процессов. Монопольно занимать одной задачей более 13 миллисекунд - моветон. Система отработает рестарт. На каждую задачу отводится 1 миллисекунда, потом обрабатывается другая задача. Иногда нельзя допустить перерыва в обработке ... впрочем это азы - отправляю к учебнику.
Вот так выглядит инициализация RTOS и разбиение нашего функционала на задачи. Некоторая особенность реализации вызвана ардуиновским делением файла main.cpp на setup() и loop().

##### файл main.cpp
```c++
#include "mboard.h"
#include "mcommands.h"
#include "mdisplay.h"
#include "mtools.h"
#include "mdispatcher.h"
#include "mconnmng.h"
#include "mmeasure.h"
#include "connectfsm.h"

static MBoard      * Board      = 0;
static MDisplay    * Display    = 0;
static MTools      * Tools      = 0;
static MCommands   * Commands   = 0;
static MMeasure    * Measure    = 0;
static MDispatcher * Dispatcher = 0;
static MConnect    * Connect    = 0;

void connectTask ( void * );
void displayTask ( void * );
void coolTask    ( void * );
void mainTask    ( void * );
void measureTask ( void * );
void driverTask  ( void * );

void setup()
{
  Display    = new MDisplay();
  Board      = new MBoard(Display);
  Tools      = new MTools(Board, Display);
  Commands   = new MCommands(Board);
  Measure    = new MMeasure(Tools);
  Dispatcher = new MDispatcher(Tools);
  Connect    = new MConnect(Tools);

  // Выделение ресурсов для каждой задачи: память, приоритет, ядро.
  // Все задачи исполняются ядром 1, ядро 0 выделено для радиочастотных задач - BT и WiFi.
  xTaskCreatePinnedToCore ( connectTask, "Connect", 10000, NULL, 1, NULL, 1 );
  xTaskCreatePinnedToCore ( mainTask,    "Main",    10000, NULL, 2, NULL, 1 );
  xTaskCreatePinnedToCore ( displayTask, "Display",  5000, NULL, 2, NULL, 1 );
  xTaskCreatePinnedToCore ( coolTask,    "Cool",     1000, NULL, 2, NULL, 1 );
  xTaskCreatePinnedToCore ( measureTask, "Measure",  5000, NULL, 2, NULL, 1 );
  xTaskCreatePinnedToCore ( driverTask,  "Driver",   5000, NULL, 2, NULL, 1 );
}

void loop() {}                            // Это тоже задача, пустая в данном случае

// Задача подключения к WiFi сети (полностью заимствована как есть)
void connectTask( void * )
{
  while(true)
  {
    Connect->run();
    // Период вызова задачи задается в TICK'ах, TICK по умолчанию равен 1мс.
    vTaskDelay( 10 / portTICK_PERIOD_MS );
  }
  vTaskDelete( NULL );
}

// Задача выдачи данных на дисплей
void displayTask( void * )
{
  while(true)
  {
    Display->runDisplay(
                        Board->Overseer->getCelsius(),
                        Tools->getAP() );
    vTaskDelay( 250 / portTICK_PERIOD_MS );
  }
  vTaskDelete( NULL );
}

// Задача управления системой теплоотвода.
void coolTask( void * )
{
  while (true)
  {
    Board->Overseer->runCool();
    vTaskDelay( 200 / portTICK_PERIOD_MS );
  }
  vTaskDelete( NULL );
}

// Задача обслуживает выбор режима работы и
// управляет конечным автоматом выбранного режима вплоть да выхода из него
void mainTask ( void * )
{
  while (true)
  {
    // Выдерживается период запуска для вычисления амперчасов. Если прочие задачи исполняются в     // порядке очереди, то эта точно по таймеру - через 0,1с.
    portTickType xLastWakeTime = xTaskGetTickCount();
    Dispatcher->run();
    vTaskDelayUntil( &xLastWakeTime, 100 / portTICK_PERIOD_MS );    // период 0,1с
  }
  vTaskDelete( NULL );
}

// Задача управления измерениями
void measureTask( void * )
{
  while (true)
  {
    Measure->run();
    vTaskDelay( 10 / portTICK_PERIOD_MS );
  }
  vTaskDelete(NULL);
}

void driverTask( void * )
{
  while (true)
  {
    Commands->doCommand();
    vTaskDelay( 100 / portTICK_PERIOD_MS );
  }
  vTaskDelete(NULL);
}
```
  Согласитесь - ничего сложного. Прототипирование выполнялось под Arduino.

Всё. Успехов!
О.В.


urk2t@yandex.ru                                Версия от  28 января 2021 года                                             molvas66@mail.ru
